---
title: فضانام‌ها و ایزوله کردن پردازه‌ها در لینوکس — قسمت اول
description: بررسی UTS، PID، Mount و Network برای ایزوله‌سازی پردازه‌ها در لینوکس با unshare/chroot و نمونه‌دستورات عملی.
slug: فضانام‌ها-در-لینوکس-۱
date: 2021-03-02 13:04:26+0000
image: cover.png
categories:
    - DevOps
    - Security
tags:
    - Linux
    - Containerization
    - Devops
    - Security
weight: 1
---

<div dir="rtl">

در این نوشتار می‌خواهیم چگونگی ایزوله کردن یک پردازه در سیستم‌عامل لینوکس را بررسی کنیم. اگر با Docker یک کانتینر ایجاد کرده باشید، در زمان کار با آن احساس کرده‌اید که داخل آن مشابه محیط یک ماشین مجازی است. مثلاً متوجه شده‌اید که از دید یک پردازه‌ی در حال اجرا در داخل کانتینر، فهرست پردازه‌ها، شبکه و فایل‌سیستم با ماشین میزبان متفاوت است، همچنین میزان منابعی مانند RAM و CPU از دید پردازه‌های داخل کانتینر محدودتر از کل مقدار در اختیار میزبان می‌باشد. 

این سطح از ایزوله‌کردن در سیستم‌عامل لینوکس توسط ویژگی‌های **namespace (فضانام)**، **cgroups**, **pivot_root** و ... پیاده‌سازی می‌شود. ایزوله کردن پردازه‌ها موضوع کلیدی در مورد نحوه‌ی عمل‌کرد کانتینرها می‌باشد. فضا‌نام‌ها منابعی را به کانتینر اختصاص می‌دهند که کاملاً مستقل از میزبان (host) و بقیه‌ی کانتینر‌های مجاور می‌باشد.

این نوشتار در دو قسمت تنظیم شده است که در ادامه **قسمت اول** آن را می‌خوانید. قسمت دوم این مطلب را می‌توانید از [اینجا](/p/فضانامها-در-لینوکس-۲/) دنبال کنید.

## فضا‌نام‌ها (namespaces) در لینوکس

فضانام یک ابزار برای محدود کردن دید پردازه‌ها نسبت به ماشین میزبان است. در kernel لینوکس فضانام‌های مختلفی وجود دارد که استفاده از هر کدام از آن‌ها اجازه می‌دهد یک پردازه در همان فضانام (مثلا شبکه) دید مختص به خود را داشته باشد. مثلاً دو پردازه‌ی در حال اجرا در میزبان می‌توانند فایل سیستم مشترکی داشته باشند ولی دید آن‌ها نسبت به کارت شبکه و تنظیمات آن متفاوت باشد و احساس کنند در دو شبکه‌ی مجزا قرار دارند. به خاطر همین سطح از ایزوله کردن، فضانام‌ها یکی از ابزارهای کلیدی برای کانتینری کردن برنامه‌ها هستند.


![*منبع: https://twitter.com/b0rk/status/1240364585766576128*](ns.jpeg)


### فضانام‌های موجود در لینوکس

<div dir=ltr>

- Unix Time Sharing (**uts**)
- Process ID (**pid**)
- Mount (**mnt**)
- Network (**net**)
- User ID / Group ID (**user**)
- Inter-Process Communications (**ipc**)
- Control Groups (**cgroup**)
- **time**

</div>

برای مشاهدهٔ فضانام‌های موجود در سیستم‌عامل می‌توانید از دستور `lsns` استفاده کنید:

```bash
sudo lsns
```

خروجی نمونه:

```
$ sudo lsns
NS TYPE   NPROCS   PID USER             COMMAND
4026531835 cgroup    405     1 root             /sbin/init splash
4026531836 pid       354     1 root             /sbin/init splash
4026531837 user      361     1 root             /sbin/init splash
4026531838 uts       397     1 root             /sbin/init splash
4026531839 ipc       397     1 root             /sbin/init splash
4026531840 mnt       385     1 root             /sbin/init splash
4026532009 net       353     1 root             /sbin/init splash
...
```

به صورت پیش‌فرض در ماشین میزبان از هر نوع فضانام یک مورد وجود دارد که توسط پردازه‌ها مورد استفاده قرار می‌گیرد. پردازه‌ها می‌توانند فضانام‌های بیشتری از نو‌ع‌های مختلفی ایجاد کنند و به آن متصل شوند. البته هر پردازه فقط می‌تواند در یک فضانام از یک نوع باشد. همچنین در صورتی که فضانام خالی باشد (مثلا هیچ‌ پردازه‌ای به آن متصل نشده باشد یا هیچ bind mount در آن وجود نداشته باشد و ...) کرنل به صورت خودکار آن را حذف می‌کند. در ادامه به بررسی فضانام‌ها می‌پردازیم.

## Unshare چیست؟

در فضای سیستم‌عامل وقتی شما درخواست اجرای یک برنامه را صادر می‌کنید، سیستم‌عامل با اجرای یک سری روال‌ها و سپس کپی‌کردن کد برنامه در حافظه‌ی اصلی (رم) یک پردازه‌ی جدید ایجاد می‌کند و به اصطلاح آن برنامه را اجرا می‌کند. در این فضا دو مفهوم پدر (parent) و فرزند (child) وجود دارد که به صورت پیش‌فرض با هم داده‌های مشترکی دارند (برای اطلاعات بیشتر به [fork](https://man7.org/linux/man-pages/man2/fork.2.html) و [clone](https://man7.org/linux/man-pages/man2/clone.2.html) مراجعه کنید). برای ایزوله‌ کردن پردازه‌ها و حذف این نقاط مشترک می‌توان از یک systemcall به نام unshare استفاده کرد. در ‌واقع با استفاده از unshare می‌توان یک برنامه را با فضانام‌هایی مختص به خودش که با پدرش یا دیگر پردازه‌های در حال اجرا مشترک نباشد، اجرا نمود.

اگر علاقه‌مند هستید جزئیات بیشتری بدانید راجع به [set_ns](https://man7.org/linux/man-pages/man2/setns.2.html) [،clone](https://man7.org/linux/man-pages/man2/clone.2.html) [،nsenter](https://man7.org/linux/man-pages/man1/nsenter.1.html) و [pivot_root](https://man7.org/linux/man-pages/man2/pivot_root.2.html) مطالعه کنید.

## فضانام UTS

هر ماشین دارای یک شناسه‌ است که به آن hostname گفته می‌شود. درست مانند URL هر وب‌سایت از این شناسه می‌توان برای اشاره به آن ماشین در داخل آن ماشین یا شبکه استفاده کرد. اگر یک ترمینال بر روی سیستم‌عامل لینوکس باز کنید با استفاده از دستور hostname می‌توانید مقدار hostname فعلی را ببینید:

```bash
$ hostname
Milad-PC
```
با قرار دادن یک پردازه در یک فضانام UTS اختصاصی، پردازه صاحب hostname و domainname اختصاصی خواهد شد و می‌توان مقادیر دلخواهی برای hostname و domainname برای آن پردازه تنظیم کرد که این تغییر تاثیری در مقادیر مربوطه در ماشین میزبان نخواهد داشت.

تکنولوژی‌های کانتینری‌کردن (مانند داکر) به هر کانتینر یک شناسه‌ی تصادفی اختصاص می‌دهند که به صورت پیش‌فرض همین شناسه به عنوان hostname در آن کانتینر استفاده می‌شود. برای بررسی این موضوع با استفاده از دستور زیر می‌توان یک کانتینر ایجاد کرد و مقدار hostname داخل کانتینر را مشاهده نمود.

```bash
$ docker run --rm -it --name hostname-test hub.hamdocker.ir/library/alpine sh
/ # hostname
a6b9c92876b7
```

با استفاده از unshare می‌توانیم یک فضانام جدید UTS ایجاد کنیم و این مورد را آزمایش کنیم. برای اینکار باید unshare را با دسترسی کاربر root اجرا کنیم. در زمان فراخوانی unshare نوع فضانام و نام‌ برنامه‌ای که می‌خواهیم اجرا کنیم را باید وارد کنیم. در صورتی که نام برنامه وارد نشود به صورت پیش‌فرض از مقدار {SHELL}$ استفاده خواهد شد.

```bash
$ sudo unshare --uts /bin/bash
root@Milad-PC:~# hostname
Milad-PC
root@Milad-PC:~# hostname xyz-hostname
root@Milad-PC:~# hostname
xyz-hostname
root@Milad-PC:~# exit
exit
$ hostname
Milad-PC
```

این کار باعث شده که bash داخل یک پردازه‌ی جدید که فضانام UTS مختص به خودش را دارد اجرا شود. هر برنامه‌ای که در این شل اجرا شود نیز این فضانام را به ارث خواهد برد. برای مثال همان‌طور که در بالا می‌بینید با اجرای hostname ابتدا همان مقدار تنظیم شده در ماشین میزبان نمایش داده می‌شود، سپس با تنظیم یک مقدار جدید، hostname در این فضانام تغییر می‌کند ولی در میزبان تغییر اعمال نشده است. این ایزوله کردن باعث می‌شود که میزبان و مهمان بدون تأثیر بر روی همدیگر مقادیر hostname را به مقداری دلخواه تغییر دهند.



## فضانام شناسهٔ پردازه‌ها (PID)

اگر شما دستور ps را داخل یک کانتینر اجرا کنید فقط پردازه‌های در حال اجرا در همان کانتینر را می‌بینید و به فهرست پردازه‌های در حال اجرا در ماشین میزبان دسترسی نخواهید داشت.

```bash
$ docker run --rm -it --name pid-test hub.hamdocker.ir/library/alpine sh
/ # ps -eaf
PID   USER     TIME  COMMAND
1 root      0:00 sh
7 root      0:00 ps -eaf
```
در ‌واقع با بهره‌برداری از فضانام PID قابلیت محدود کردن امکان دسترسی به پردازه‌های در حال اجرا در میزبان برای کانتینرها با ایزوله کردن فهرست شناسه‌ی پردازه‌ها فراهم می‌شود. برای آزمایش فضانام PID مجدداً از unshare استفاده می‌کنیم:

```bash
$ sudo unshare --pid sh
# id
uid=0(root) gid=0(root) groups=0(root)
# id
sh: 2: Cannot fork
# ls
sh: 3: Cannot fork
# id
sh: 4: Cannot fork
```

اگه به خروجی دستورات بالا دقت کنید به نظر مشکلی وجود دارد. به غیر از دستور اول، بقیه دستور‌ها با خطا مواجه شده‌اند. اگه به متن خطا دقت کنید قالب آن از چپ به راست به صورت نام دستور، شناسه‌ی پردازه و متن خطا می‌باشد. با اجرای‌ هر دستور، شناسه‌‌ی پردازه‌ها در حال افزایش است و این نشان دهنده‌ی اعمال ‌شدن فضانام PID است، اما اگر نتوان بیش از یک پردازه در این فضانام اجرا کرد، کاملاً بلااستفاده باقی می‌ماند. متن خطای دریافتی و همچنین توضیحات unshare نشان می‌دهند که:

```bash
$ man unshare
-f, --fork
Fork  the  specified  program as a child process of unshare rather than running it directly.  This is useful when creating a new PID namespace.
```

پس با استفاده از fork-- پردازه‌ی جدید مستقیماً به صورت یک فرزند از unshare اجرا خواهد شد. حال مجدداً یک فضانام PID ایجاد می‌کنیم. با اجرای چند دستور مشاهده می‌کنیم خطای قبل رفع شده است:

```bash
$ sudo unshare --pid --fork /bin/bash
root@Milad-PC:/tmp# id
uid=0(root) gid=0(root) groups=0(root)
root@Milad-PC:/tmp# cat /etc/hostname
Milad-PC
```
سپس برای فهرست کردن پردازه‌های موجود در فضانام جدید از دستور ps استفاده می‌کنیم:

```bash
root@Milad-PC:/tmp# ps -eaf
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 فوریه18 ? 00:00:09 /sbin/init splash
root         2     0  0 فوریه18 ? 00:00:00 [kthreadd]
root         4     2  0 فوریه18 ? 00:00:00 [kworker/0:0H]
...
root      6210  9375  0 02:02 pts/2    00:00:00 sudo unshare --pid --fork /bin/bash
root      6211  6210  0 02:02 pts/2    00:00:00 unshare --pid --fork /bin/bash
root      6212  6211  0 02:02 pts/2    00:00:00 /bin/bash
root      6421  6212  0 02:03 pts/2    00:00:00 ps -eaf
…
milad     7089     1  0 فوریه18 ? 00:00:06 /lib/systemd/systemd --user
milad     7090  7089  0 فوریه18 ? 00:00:00 (sd-pam)
milad     8536  8426  1 فوریه18 tty2 01:24:19 /usr/lib/chromium-browser/chromium-browser --type=renderer --field-trial-handle=1178073432432703370
milad    10327  7677  0 فوریه18 pts/1 00:00:11 /usr/bin/zsh
```

اگر در یک کانتینر ایجاد شده مثلاً با داکر دستور ps را وارد کنید فقط پردازه‌های موجود در همان کانتینر را مشاهده خواهید کرد. با توجه به خروجی دستور ps شاید این‌طور به نظر برسد که فضانام PID جدید به درستی پردازه‌ی جدید را ایزوله نکرده و پردازه‌ی جدید به فهرست پردازه‌های در حال اجرا در میزبان دسترسی دارد اما در‌ واقع این‌طور نیست. فهرست پردازه‌های میزبان به این علت در دسترس است که دستور ps اطلاعات را از روی فایل‌های موجود در مسیر proc/ می‌خواند و پردازش می‌کند. اگه از مسیر proc/ در میزبان ls بگیرید فهرستی از دایرکتوری‌ها را مشاهده خواهید کرد که متناظر با پردازه‌های در حال اجرا در سیستم‌عامل می‌باشد که در داخل آن‌ها اطلاعات کامل پردازه‌ها به صورت فهرستی از فایل‌ها وجود دارد.

```bash
$ ls /proc
1      132    1483   1943   2052   239    2566   272    321   4510  513   540   6635  7433  7575  817   9136           fs           pagetypeinfo
…
```

برای اینکه در فضانام جدید دستور ps فقط اطلاعات پردازه‌های موجود در همین فضانام را برگردانند باید یک مسیر proc/ مجزا برای همین فضانام وجود داشته باشد که کرنل اطلاعات پردازه‌های موجود در آن را مدیریت کند که برای پیاده‌سازی آن می‌توان از chroot استفاده کرد.

در زمان ایجاد یک کانتینر به طور پیش‌فرض به جای دسترسی کامل به فایل‌سیستم میزبان، پردازه‌ها به بخش کوچکی از فایل سیستم دسترسی خواهند داشت، چون درست بعد از ایجاد کانتینر دایرکتوری root برای پردازه‌ی اصلی داخل کانتینر تغییر می‌کند. در سیستم‌عامل لینوکس این عملیات با استفاده از chroot انجام می‌شود.

در توضیحات chroot اشاره شده است که chroot یک دستور یا shell را در دایرکتوری جدید اجرا می‌کند و همچنین اگر دستوری وارد نشود به صورت پیش‌فرض از مقدار متغیر SHELL$ استفاده خواهد کرد.

```bash
$ man chroot
NAME
chroot - run command or interactive shell with special root directory
…
If no command is given, run '&quot$SHELL&quot -i' (default: '/bin/sh -i').
```

برای درک بهتر این فرآیند دستورات زیر را اجرا می‌کنیم:

```
$ mkdir /tmp/new_root_dir
$ sudo chroot /tmp/new_root_dir
chroot: failed to run command ‘/usr/bin/zsh’: No such file or directory
$ sudo chroot /tmp/new_root_dir id
chroot: failed to run command ‘id’: No such file or directory
$ sudo chroot /tmp/new_root_dir ls
chroot: failed to run command ‘ls’: No such file or directory
```

با توجه به متن‌ خطا به نظر می‌رسد که بعد از تغییر دایرکتوری root دسترسی به برنامه‌های موجود در دایرکتوری bin/ امکان‌پذیر نیست. بنابراین حتی امکان استفاده از دستوراتی مثل ls و id فراهم نیست. بنابراین تمامی فایل‌‌های مورد نیاز، باید به دایرکتوری جدید انتقال داده شود؛ کاملاً مشابه زمانی که یک کانتینر واقعی ایجاد می‌شود. یک کانتینر از روی یک image ایجاد می‌شود که آن image حاوی تمامی فایل‌هایی است که پردازه‌های موجود در آن کانتینر به آن نیاز دارند. در‌ واقع آن image دقیقاً حاوی فایل‌سیستمی است که پردازه‌ها آن را می‌بینند.

بنابراین اگر ما فایل‌سیستم یک سیستم‌عامل کوچک مثل Alpine Linux را داشته باشیم‌ می‌توانیم به آن chroot کنیم و آن را در فضانام جدید اجرا کنیم. برای انجام این کار دستورات زیر را اجرا می‌کنیم که ابتدا یک دایرکتوری جدید ایجاد می‌شود، سپس آخرین نسخه‌ی فعلی فایل‌سیستم را دانلود و استخراج می‌کنیم.

```bash
$ mkdir alpine
$ cd alpine
$ curl -o alpine.tar.gz http://dl-cdn.alpinelinux.org/alpine/v3.13/releases/x86_64/alpine-minirootfs-3.13.0-x86_64.tar.gz
% Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
Dload  Upload   Total   Spent    Left  Speed
100 2664k  100 2664k    0     0  15768      0  0:02:53  0:02:53 --:--:-- 18207
$ tar -xzf alpine.tar.gz
$ ls
alpine.tar.gz  bin  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
$ rm -rf alpine.tar.gz

```

سپس با chroot در این دایرکتوری می‌توانیم دستورات را در مسیر جدید اجرا کنیم.


```bash
$ cd ..
$ sudo chroot alpine ls
bin    dev    dst    etc    home   lib    media  mnt    opt    proc   root   run    sbin   src    srv    sys    tmp    usr    var ...
```

با اتمام اجرای پردازه (یا دستور) فرزند، مجدداً کنترل به پردازه‌ی پدر برمی‌گردد. برای اینکه بتوانیم دستورات بیشتری در دایرکتوری جدید اجرا کنیم می‌توانیم یک شل را به عنوان پردازه‌ی فرزند اجرا کنیم.

```
$ sudo chroot alpine sh
/ # id
uid=0(root) gid=0(root) groups=0(root)
/ # ls
bin    dev    dst    etc    home   lib    media  mnt    opt    proc   root   run    sbin   src    srv    sys    tmp    usr    var
/ # exit
```

حالا با بهره‌برداری از قابلیت chroot و فضانام‌ها می‌توانیم یک فضانام PID ایجاد کرده و در زمان فراخوانی unshare دستور chroot را اجرا کنیم و سپس در داخل فضانام دایرکتوری proc/ فایل‌سیستم (image) را mount کنیم. با انجام این گام‌ها یک قدم در ایزوله کردن پردازه‌های داخل کانتینر برداشته می‌شود و جزئیات پردازه‌های میزبان از دید پردازه‌های در حال اجرا در فضانام مخفی خواهد بود. برای آزمایش این موضوع دستورات زیر را اجرا می‌کنیم:

```bash
$ sudo unshare --pid --fork chroot alpine sh
/ # mount -t proc proc proc
/ # ps -eaf
PID   USER     TIME  COMMAND
1 root      0:00 sh
3 root      0:00 ps -eaf
```

## فضانام Mount

برای اینکه پردازه‌های داخل یک کانتینر به فایل سیستم میزبان دسترسی نداشته باشد، باید بین آن‌ها مرزی ایجاد شود. با استفاده از فضانام mount می‌توان این قابلیت را برای پردازه‌ها فراهم کرد.

برای مثال در زیر ما در یک فضانام mount جدید یک bind mount ایجاد می‌کنیم. Bind mounts این قابلیت را فراهم می‌کند که بخشی از یک فایل سیستم که قبلاً mount شده را در یک دایرکتوری دیگه مجدداً mount کنید. با بهره‌برداری از این قابلیت می‌شود عملیات مختلفی مثل اشتراک گذاری فقط خواندنی، chroot jail یا کانتینری کردن را پیاده‌سازی کرد.


```
$ sudo unshare --mount /bin/bash
root@Milad-PC:~# mkdir src
root@Milad-PC:~# echo &quotsample&quot > src/test.txt
root@Milad-PC:~# ls src
test.txt
root@Milad-PC:~# mkdir dst
root@Milad-PC:~# ls dst
root@Milad-PC:~# mount --bind src dst
root@Milad-PC:~# ls dst
test.txt
root@Milad-PC:~# cat dst/test.txt
sample
```

همان‌طور که می‌بینیم بعد از اعمال bind محتوای دایرکتوری src در dst نیز در دسترس است. با استفاده از findmnt می‌توانیم جزئیات بیشتری را ببینیم که این mount در این فضانام ایجاد شده است و از دید میزبان مخفی می‌باشد.

```bash
root@Milad-PC:~# findmnt dst
TARGET          SOURCE                FSTYPE OPTIONS
/home/milad/dst /dev/sda5[/milad/src] ext4   rw,relatime,data=ordered

```

برای مثال در یک ترمینال دیگر در میزبان و خارج از این فضانام همین دستور را اجرا می‌کنیم.

```bash
$ findmnt dst
```

حالا اگر همین دستور findmnt را در فضانام ساخته شده مجدداً بدون پارامتر اجرا کنیم فهرست کاملی از mountهای میزبان را مشاهده خواهیم کرد. همان‌طور که در مورد فضانام PID اشاره کردیم، انتظار داریم این موارد از دید یک کانتینر مخفی شده باشد. مشابه شناسه‌ی پردازه‌ها کرنل از مسیر proc/ID/mounts/ اطلاعات مربوط به mountهای هر پردازه را می‌خواند. بنابراین وقتی که یک پردازه با یک فضانام اختصاصی ایجاد کنیم ولی همچنان از مسیر proc/ میزبان استفاده کند، پردازه می‌تواند به اطلاعات میزبان دسترسی داشته باشد.

برای ایزوله کردن یک پردازه‌ نیاز است که ساخت یک فضانام جدید همراه با ایجاد یک فایل‌سیستم root و یک proc mount انجام شود.

```bash
$ sudo unshare --mount chroot alpine sh
/ # ls
bin    dev    etc    home   lib    media  mnt    opt    proc   root   run    sbin   srv    sys    tmp    usr    var
/ # mount -t proc proc proc
/ # mount
proc on /proc type proc (rw,relatime)
/ # mkdir src
/ # echo &quotsample&quot > src/test.txt
/ # mkdir dst
/ # mount --bind src dst
/ # ls dst
test.txt
/ # cat dst/test.txt
sample
/ # mount
proc on /proc type proc (rw,relatime)
/dev/sda1 on /dst type ext4 (rw,relatime,errors=remount-ro,data=ordered)
/ #
```

به طور مشابه وقتی شما یک دایرکتوری در سیستم میزبان را در یک کانتینر mount می‌کنید (مثلاً در داکر: docker run -v path_in_host:path_in_container) ابتدا فایل‌سیستم root کانتینر در محل مناسب قرار می‌گیرد. سپس دایرکتوری مقصد در کانتینر ایجاد شده و در نهایت دایرکتوری مبدأ در مقصد bind mount می‌شود.

## فضانام شبکه (Network)

یکی دیگر از سطوح ایزوله کردن، مربوط به فضای شبکه می‌باشد. فضانام شبکه به پردازه‌ها اجازه می‌دهد که دید اختصاصی نسبت به کارت شبکه و جدول مسیریابی داشته باشند.

با استفاده از دستور زیر می‌توان یک فضانام شبکه ایجاد کرد.

```bash
$ sudo unshare –net /bin/bash
```

با استفاده از دستور lsns و تعیین نوع net می‌توانیم جزئیات فضانام ایجاد شده را ببینیم. شناسه‌ی پردازه را به خاطر بسپارید.

```bash
root@Milad-PC:/tmp# lsns -t net
NS TYPE NPROCS   PID USER  COMMAND
...
4026532516 net       2  5268 root  /bin/bash
...
```

به صورت پیش‌فرض وقتی که یک پردازه را در فضانام شبکه‌ی مختص به خودش قرار می‌دهیم، فقط یک کارت شبکه loopback دارد.


```bash
root@Milad-PC:~# ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

برای اینکه پردازه‌ی در حال اجرا در کانتینر بتواند انتقال داده داشته باشد باید یک کارت شبکه‌ی مجازی ایجاد کنیم که مثل دو سر یک کابل شبکه عمل می‌کند. در‌ واقع فضانام شبکه‌ی اختصاصی ایجاد شده برای پردازه را به فضانام شبکه میزبان (پیش‌فرض) متصل می‌کند. برای این کار در یک ترمینال دیگر دستور زیر را وارد می‌کنیم:


```bash

$ ip link add myeth1 netns 5268 type veth peer name myeth2 netns 1
```

اگر بخواهیم دستور بالا را به زبان ساده بیان کنیم، ip link add اعلام می‌کند که می‌خواهیم یک اتصال ایجاد کنیم که در فضانام مربوط به پردازه‌ی ۵۲۶۸ یک کارت شبکه‌ی مجازی به اسم myeth1 ایجاد کند و آن را به فضانام پردازه‌ی ۱ وصل کند و نام کارت شبکه‌ی مجازی را myeth2 تنظیم کند.

حالا اگر در یک ترمینال در میزبان و همچنین در داخل فضانام جدید فهرست کارت شبکه‌ها را بررسی کنیم این دو کارت شبکه را خواهیم دید.

در داخل فضانام شبکه‌ی جدید:

```bash
root@Milad-PC:/tmp# ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: myveth1@if18: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000 link/ether 5e:f7:93:bf:19:80 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

در داخل میزبان:

```bash
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000 link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00 inet 127.0.0.1/8 scope host lo valid_lft forever preferred_lft forever inet6 ::1/128 scope host valid_lft forever preferred_lft forever 
…
18: myveth2@if2: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000 link/ether ea:4d:95:01:8c:db brd ff:ff:ff:ff:ff:ff link-netnsid 1
```

در حالت فعلی هر دو کارت شبکه‌ی جدید در وضعیت Down هستند و برای اینکه امکان انتقال داده فراهم شود باید وضعیت هر دو کارت به Up تغییر کند و به آن‌ها آدرس IP تخصیص داده شود.

در ترمینال میزبان:

```bash
$ sudo ip link set myveth2 up
$ sudo ip addr add 192.168.30.100/24 dev myveth2
$ ip a s myveth2
18: myveth2@if2: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN group default qlen 1000 link/ether ea:4d:95:01:8c:db brd ff:ff:ff:ff:ff:ff link-netnsid 1 inet 192.168.30.100/24 scope global myveth2 valid_lft forever preferred_lft forever
```

در فضانام جدید:

```bash

$ sudo ip link set myveth1 up
$ sudo ip addr add 192.168.30.101/24 dev myveth1
$ ip a s myveth1 
2: myveth1@if18: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000 link/ether 5e:f7:93:bf:19:80 brd ff:ff:ff:ff:ff:ff link-netnsid 0 inet 192.168.30.101/24 scope global myveth1valid_lft forever preferred_lft forever
```

همان‌طور که اشاره کردیم با ایجاد یک فضانام شبکه، جدول مسیر‌یابی و کارت شبکه‌ها ایزوله خواهد شد و پردازه‌ی موجود در کانتینر به اطلاعات میزبان دسترسی نخواهد داشت.

```bash
root@Milad-PC:/tmp# ip route 
192.168.30.0/24 dev myveth1 proto kernel scope link src 192.168.30.101
```

حالا هم در ترمینال ماشین میزبان و هم در فضانام جدید می‌توانیم اتصال را تست کنیم. در ترمینال ماشین میزبان این دستور را اجرا کنید:

```bash
$ ping 192.168.30.101
PING 192.168.30.101 (192.168.30.101) 56(84) bytes of data.
64 bytes from 192.168.30.101: icmp_seq=1 ttl=64 time=0.043 ms
64 bytes from 192.168.30.101: icmp_seq=2 ttl=64 time=0.079 ms
^C
--- 192.168.30.101 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1020ms
rtt min/avg/max/mdev = 0.043/0.061/0.079/0.018 ms
```

همین‌طور در فضانام:

```bash
root@Milad-PC:/tmp# ping 192.168.30.100
PING 192.168.30.100 (192.168.30.100) 56(84) bytes of data.
64 bytes from 192.168.30.100: icmp_seq=1 ttl=64 time=0.078 ms
64 bytes from 192.168.30.100: icmp_seq=2 ttl=64 time=0.100 ms
^C
--- 192.168.30.100 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1015ms
rtt min/avg/max/mdev = 0.078/0.089/0.100/0.011 ms
```

مطابق خروجی بالا می‌بینیم که اتصال بین دو طرف برقرار شده است. 

در قسمت اول با فضانام‌های uts، pid، mount و network آشنا شدیم. در قسمت [دوم](/p/فضانامها-در-لینوکس-۱/) این نوشتار فضانام‌های user، ipc، cgroup و time را مورد بررسی قرار داده‌ایم.


</div>
