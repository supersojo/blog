---
title: 分析下mutter窗口管理器的一个问题
date: 2019-3-1 23:01:20
tags:
- mutter
- x11
categories:
- mutter
---

## mutter problem
gnome hangup after 49.7 days.

## mutter?

mutter是GNOME3的窗口管理器，用来取代Metacity。

> https://zh.wikipedia.org/wiki/Mutter

## patch info?

[backends/x11: Fix time-comparison bug causing hang]https://gitlab.gnome.org/GNOME/mutter/-/merge_requests/12/diffs?commit_id=102abeedf4d3a804bed2e1930a93a52e4475a3bb

> A comparison in translate_device_event() does not account for the fact
> that X's clock wraps about every 49.7 days.  When triggered, this causes
> an unresponsive GUI.
> Replace simple less-than comparison with XSERVER_TIME_IS_BEFORE macro,
> which accounts for the wrapping of X's clock.

详细的提交记录
```
commit 942883577ea700f0f419c335185891ff9a02e07b
Author: Jonas Ådahl <jadahl@gmail.com>
Date:   Fri Oct 25 10:06:55 2019 +0200

    x11: Limit touch replay pointer events to when replaying

    When a touch sequence was rejected, the emulated pointer events would be
    replayed with old timestamps. This caused issues with grabs as they
    would be ignored due to being too old. This was mitigated by making sure
    device event timestamps never travelled back in time by tampering with
    any event that had a timestamp seemingly in the past.

    This failed when the most recent timestamp that had been received were
    much older than the timestamp of the new event. This could for example
    happen when a session was left not interacted with for 40+ days or so;
    when interacted with again, as any new timestamp would according to
    XSERVER_TIME_IS_BEFORE() still be in the past compared to the "most
    recent" one. The effect is that we'd always use the `latest_evtime` for
    all new device events without ever updating it.

    The end result of this was that passive grabs would become active when
    interacted with, but would then newer be released, as the timestamps to
    XIAllowEvents() would out of date, resulting in the desktop effectively
    freezing, as the Shell would have an active pointer grab.

    To avoid the situation where we get stuck with an old `latest_evtime`
    timestamp, limit the tampering with device event timestamp to 1) only
    pointer events, and 2) only during the replay sequence. The second part
    is implemented by sending an asynchronous message via the X server after
    rejecting a touch sequence, only potentially tampering with the device
    event timestamps until the reply. This should avoid the stuck timestamp
    as in those situations, we'll always have a relatively up to date
    `latest_evtime` meaning XSERVER_TIME_IS_BEFORE() will not get confused.

    https://gitlab.gnome.org/GNOME/mutter/merge_requests/886
```
指针焦点被某个程序grab后，由于时间戳不对一直得不到释放。指针事件无法正确传递导致
桌面无响应。

## 代码
```
static void
translate_device_event (MetaBackendX11 *x11,
                        XIDeviceEvent  *device_event)
{
  MetaBackendX11Private *priv = meta_backend_x11_get_instance_private (x11);

  meta_backend_x11_translate_device_event (x11, device_event);

  if (!device_event->send_event && device_event->time != CurrentTime)
    {
-     if (device_event->time < priv->latest_evtime)
+     if (XSERVER_TIME_IS_BEFORE (device_event->time, priv->latest_evtime))
        {
          /* Emulated pointer events received after XIRejectTouch is received
           * on a passive touch grab will contain older timestamps, update those
           * so we dont get InvalidTime at grabs.
           */
          device_event->time = priv->latest_evtime;
        }

      /* Update the internal latest evtime, for any possible later use */
      priv->latest_evtime = device_event->time;
    }
}
```
time是32位无符号整数，单位是ms，大约49.7天会溢出。在溢出后，导致事件的时间戳
被设置位很久之前的值，在提交给xserver时无效。

## XSERVER_TIME_IS_BEFORE 

```
/* Xserver time can wraparound, thus comparing two timestamps needs to take
 * this into account.  Here's a little macro to help out.  If no wraparound
 * has occurred, this is equivalent to
 *   time1 < time2
 * Of course, the rest of the ugliness of this macro comes from accounting
 * for the fact that wraparound can occur and the fact that a timestamp of
 * 0 must be special-cased since it means older than anything else.
 *
 * Note that this is NOT an equivalent for time1 <= time2; if that's what
 * you need then you'll need to swap the order of the arguments and negate
 * the result.
 */
#define XSERVER_TIME_IS_BEFORE_ASSUMING_REAL_TIMESTAMPS(time1, time2) \
	( (( (time1) < (time2) ) && ( (time2) - (time1) < ((guint32)-1)/2 )) || \
		(( (time1) > (time2) ) && ( (time1) - (time2) > ((guint32)-1)/2 )) \
	)
#define XSERVER_TIME_IS_BEFORE(time1, time2) \
	( (time1) == 0 || \
		(XSERVER_TIME_IS_BEFORE_ASSUMING_REAL_TIMESTAMPS(time1, time2) && \
		(time2) != 0) \
	)
```
根据patch的修改，如果时间戳不对，则窗口不响应。


X服务器对时间戳的使用例子：
```
time = ClientTimeToServerTime(ctime);
if ((CompareTimeStamps(time, currentTime) == LATER) ||
             (CompareTimeStamps(time, grabInfo->grabTime) == EARLIER)
	xxx
```
这里先调用ClientTimeToServerTime把32位无符号转换为
TimeStamp类型，然后和currentTime进行比较。

## mutter程序
```
main
 \--meta_init
      \--- meta_clutter_init
	             \-- source = g_source_new (&event_funcs, sizeof (GSource));
				 \-- meta_backend_post_init
				         \-- META_BACKEND_GET_CLASS (backend)->post_init (backend);
						       \-- x_event_source_new
							          \--  source = g_source_new (&x_event_funcs, sizeof (XEventSource));
      \--- meta_main_loop = g_main_loop_new (NULL, FALSE);
 \--meta_run 
      \--- g_main_loop_run (meta_main_loop);

```
mutter是基于glib的程序，通过事件源机制注册事件处理函数。

meta_backend_post_init中创建的事件源处理函数会调用XPending把窗口事件取出来，然后dispatch。
mutter作为窗口管理器会"收到"很多窗口的事件(实际也是通过xserver发过来，见[substructure redirection](https://jichu4n.com/posts/how-x-window-managers-work-and-how-to-write-one-part-i/))，主要是窗口事件，这样mutter负责把窗口消息再发给
X服务器。



## CurrentTime
```
/usr/include/X11/X.h:139:
#define CurrentTime          0L        /* special Time */
```
xserver如果使用client传递过来的timestamp
```
./dix/dixutils.c
/*
 * convert client times to server TimeStamps
 */
#define HALFMONTH ((unsigned long) 1<<31)
TimeStamp
ClientTimeToServerTime(CARD32 c)
{
    TimeStamp ts;

    if (c == CurrentTime)
        return currentTime;
    ts.months = currentTime.months;
    ts.milliseconds = c;
    if (c > currentTime.milliseconds) {
        if (((unsigned long) c - currentTime.milliseconds) > HALFMONTH)
            ts.months -= 1;
    }
    else if (c < currentTime.milliseconds) {
        if (((unsigned long) currentTime.milliseconds - c) > HALFMONTH)
            ts.months += 1;
    }
    return ts;
}
```

## xserver如何更新currentTime
```
./dix/dispatch.c
void
UpdateCurrentTime(void)
{
    TimeStamp systime;

    /* To avoid time running backwards, we must call GetTimeInMillis before
     * calling ProcessInputEvents.
     */
    systime.months = currentTime.months;
    systime.milliseconds = GetTimeInMillis();
    if (systime.milliseconds < currentTime.milliseconds)
        systime.months++;
    if (InputCheckPending())
        ProcessInputEvents();
    if (CompareTimeStamps(systime, currentTime) == LATER)
        currentTime = systime;
}
```

## references
1. [x basics](https://magcius.github.io/xplain/article/x-basics.html)
2. [x concepts](https://www.x.org/wiki/guide/concepts/)
3. [X Window System Protocol](https://www.x.org/docs/XProtocol/proto.pdf)
4. [How X Window Managers Work, And How To Write One](https://jichu4n.com/posts/how-x-window-managers-work-and-how-to-write-one-part-i/)