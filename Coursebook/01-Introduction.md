---
link-citations: true
title: "**CS341 系统编程课程手册**"
---

- [[#^introduction|引言]]
  - [[#^authors|作者]]


# 引言 ^introduction

**致未来的幸福孩子们：过去的人们向你们致以问候。** — **母校校训（Alma Mater）**

在伊利诺伊大学厄巴纳-香槟分校，我们从根本上相信：我们有权让这所大学对所有未来的学生都变得更好。这一信念铭刻在我们的母校校训之中，也构成了课程团队精神的内核。因此，我们编写了这本课程手册。它是一本自由开放的系统编程教材，任何人都可以阅读它、为它贡献内容，并且可以永久地修改它。我们不认为信息应当被关在围墙花园之中；我们真心相信，复杂的概念可以被简单地、完整地解释清楚，让任何人都能理解。本书的目标是教你系统编程的基础知识，并让你对它的复杂性形成一些直觉。

和任何一本好书一样，它还不完善。我们仍有大量示例、想法、排版错误和章节需要打磨。如果你发现任何问题，请提交一个
<a href="https://github.com/illinois-cs241/coursebook/issues">https://github.com/illinois-cs241/coursebook/issues</a>
issue，或者把发现的错别字清单邮件发给
<a href="https://cs341.cs.illinois.edu/staff">https://cs341.cs.illinois.edu/staff</a>，我们很乐意去处理。我们始终在努力把这本书变得更好——无论是对一年之后还是十年之后的学生。

本作品基于原版
<a href="https://github.com/angrave/SystemProgramming/wiki">https://github.com/angrave/SystemProgramming/wiki</a>。
所有这些人的辛勤工作都体现在下面的章节里。

哦，还有那只鸭子？一直读到「同步」那一章再说吧 :)。

再次感谢，祝你阅读愉快！

– Bhuvy

## 作者 ^authors

``` markdown
Bhuvan Venkatesh <bhuvan.venkatesh21@gmail.com>
Lawrence Angrave <angrave@illinois.edu>
joebenassi <joebenassi@gmail.com>
jakebailey <zikaeroh@gmail.com>
Ebrahim Byagowi <ebrahim@gnu.org>
Alex Kizer <the.alex.kizer@gmail.com>
dimyr7 <dimyr7.puma@gmail.com>
Ed K <ed.karrels@gmail.com>
ace-n <nassri2@illinois.edu>
josephmilla <jjtmilla@gmail.com>
Thomas Liu <thomasliu02@gmail.com>
Johnny Chang <johnny@johnnychang.com>
goldcase <johnny@johnnychang.com>
vassimladenov <vassi1995@icloud.com>
SurtaiHan <surtai.han@gmail.com>
Brandon Chong <bchong95@users.noreply.github.com>
Ben Kurtovic <ben.kurtovic@gmail.com>
dprorok2 <dprorok2@illinois.edu>
anchal-agrawal <aagrawa4@illinois.edu>
daeyun <daeyunshin@gmail.com>
bchong95 <bschong2@illinois.edu>
rushingseas8 <georgealeks@hotmail.com>
lukspdev <lllluuukke@gmail.com>
hilalh <habashi2@illinois.edu>
dimyr7 <dimyr7@hotmail.com>
Azrakal <genxswordsman@hotmail.com>
G. Carl Evans <gcevans@gmail.com>
Cornel Punga <cornel.punga@gmail.com>
vikasagartha <vikasagartha@gmail.com>
dyarbrough93 <dyarbrough93@yahoo.com>
berwin7996 <berwin7996@gmail.com>
Sudarshan Govindaprasad <SudarshanGp@users.noreply.github.com>
NMyren <ntmyren@gmail.com>
Ankit Gohel <ankitgohel1996@gmail.com>
vha-weh-shh <bhaweshchhetri1@gmail.com>
sasankc <sasank.chundi@gmail.com>
rishabhjain2795 <rishabhjain2795@gmail.com>
nickgarfield <nickgarfield@icloud.com>
by700git <aaabox@yeah.net>
bw-vbnm <bwang19@illinois.edu>
Navneeth Jayendran <jayndrn2@illinois.edu>
Joe Benassi <joebenassi@gmail.com>
Harpreet Singh <hshssingh4@gmail.com>
FenixFeather <thomasliu02@gmail.com>
EntangledLight <bdelapor@illinois.edu>
Bliss Chapman <bliss.chapman@gmail.com>
zikaeroh <zikaeroh@gmail.com>
time bandit <radicalrafi@gmail.com>
paultgibbons <paultgibbons@gmail.com>
kevinwang <kevin@kevinwang.com>
cPolaris <cPolaris@users.noreply.github.com>
Zecheng (張澤成) <zzhan147@illinois.edu>
Wieschie <supernova190@gmail.com>
WeiL <z920631580@gmail.com>
Graham Dyer <gdyer2@illinois.edu>
Arun Prakash Jana <engineerarun@gmail.com>
Ankit Goel <ankitgoel616@gmail.com>
Allen Kleiner <akleiner24@gmail.com>
Abhishek Deep Nigam <adn5327@users.noreply.github.com>
zmmille2 <zmmille2@gmail.com>
sidewallme <sidewallme@gmail.com>
raych05 <raymondcheng05@gmail.com>
mmahes <malinixmahes@gmail.com>
mass <amass1212@gmail.com>
kovaka <jakelagrou@gmail.com>
gmag23 <gmag23@gmail.com>
ejian2 <ejian2@illinois.edu>
cerutii <marc.ceruti@gmail.com>
briantruong777 <briantruong777@gmail.com>
adevar <adevar2@illinois.edu>
Yuxuan Zou (Sean) <yzouac@connect.ust.hk>
Xikun Zhang <xikunz2@illinois.edu>
Vishal Disawar <disawar2@illinois.edu>
Taemin Shin <cprayer@naver.com>
Sujay Patwardhan <sujay.patwardhan@gmail.com>
SufeiZ <sufeizhang92@gmail.com>
Sufei Zhang <sufeizhang92@gmail.com>
Steven Shang <sstevenshang@users.noreply.github.com>
Steve Zhu <st.zhu1@gmail.com>
Sibo Wang <sibowsb@gmail.com>
Shane Ryan <shane1027@users.noreply.github.com>
Scott Bigelow <epheph@gmail.com>
Riyad Shauk <riyadshauk@users.noreply.github.com>
Nathan Somers <nsomers2@illinois.edu>
LieutenantChips <vkaraku2@illinois.edu>
Jacob K LaGrou <jakelagrou@gmail.com>
George <ruan3@illinois.edu>
David Levering <dmlevering@gmail.com>
Bernard Lim <bernlim93@users.noreply.github.com>
zwang180 <zshwang0809@gmail.com>
xuanwang91 <LilyBiology2010@gmail.com>
xin-0 <xintong2@illinois.edu>
wchill <wchill1337@gmail.com>
vishnui <vishnui@gmail.com>
tvarun2013 <tvarun2013@gmail.com>
sstevenshang <sstevenshang@users.noreply.github.com>
ssquirrel <lxl_zhang@Hotmail.com>
smeenai <shoaib.meenai@gmail.com>
shrujancheruku <shrujancheruku@gmail.com>
ruiqili2 <ruiqili2@users.noreply.github.com>
rchwlsk2 <rchwlsk2@illinois.edu>
ralphchung <ralphchung2005@gmail.com>
nikioftime <ncwells2@illinois.edu>
mosaic0123 <truffer@live.com>
majiasheng <jiasheng.ma@yahoo.com>
m <cheonghiuwaa@gmail.com>
li820970 <li820970@gmail.com>
kuck1 <kuck1@illinois.edu>
kkgomez2 <kkgomez2@users.noreply.github.com>
jjames34 <James_Jerry1@yahoo.com>
jargals2 <jargals2@ilinois.edu>
hzding621 <hzding621@users.noreply.github.com>
hzding621 <hzding621@gmail.com>
hsingh23 <hisingh1@gmail.com>
denisdemaisbr <denis@roo.com.br>
daishengliang <daishengliang@gmail.com>
cucumbur <bomblolism@gmail.com>
codechao999 <brianweis@comcast.net>
chrisshroba <chrisshroba@gmail.com>
cesarcastmore <cesar.cast.more@gmail.com>
briantruong777 <briantruong777@users.noreply.github.com>
botengHY <tengbo1992@gmail.com>
blapalp <pzkmmmh@gmail.com>
bchhetri1 <bhaweshchhetri1@gmail.com>
anadella96 <aisha.nadella@gmail.com>
akleiner2 <akleiner24@gmail.com>
aRatnam12 <ansh.ratnam@gmail.com>
Yash Sharma <yashosharma@gmail.com>
Xiangbin Hu <xhu27@illinois.edu>
WininWin <ezoneid@gmail.com>
William Klock <william.klock@gmail.com>
WenhanZ <marinebluee@hotmail.com>
Vivek Pandya <vivekvpandya@gmail.com>
Vineeth Puli <vpuli98@gmail.com>
Vangelis Tsiatsianas <vangelists@users.noreply.github.com>
Vadiml1024 <vadim@mbdsys.com>
Utsav2 <ukshah2@illinois.edu>
Thirumal Venkat <zapstar@users.noreply.github.com>
TheEntangledLight <bdelapor@illinois.edu>
SudarshanGp <SudarshanGp@users.noreply.github.com>
Sudarshan Konge <6025419+sudk1896@users.noreply.github.com>
Slix <slixpk@gmail.com>
Sasank Chundi <sasank.chundi@gmail.com>
SachinRaghunathan <srghnth2@illinois.edu>
Rémy Léone <remy.leone@gmail.com>
RusselLuo <russelluo@gmail.com>
Roman Vaivod <littlewhywhat@gmail.com>
Rohit Sarathy <rohit@sarathy.org>
Rick Sheahan <bomblolism@gmail.com>
Rakhim Davletkaliyev <freetonik@gmail.com>
Punitvara <punitvara@gmail.com>
Phillip Quy Le <pitlv2109@gmail.com>
Pavle Simonovic <simonov2@illinois.edu>
Paul Hindt <phindt@gmail.com>
Nishant Maniam <nishant.maniam@gmail.com>
Mustafa Altun <gmail@mustafaaltun.com>
Mohammed Sadik P. K <sadiqpkp@gmail.com>
Mingchao Zhang <43462732+mingchao-zhang@users.noreply.github.com>
Michael Vanderwater <vndrwtr2@users.noreply.github.com>
Maxiwell Luo <maxluoXIII@gmail.com>
LunaMystic <suxianghan@outlook.com>
Liam Monahan <liam@liammonahan.com>
Joshua Wertheim <joshwertheim@gmail.com>
John Pham <newhope11134@gmail.com>
Johannes Scheuermann <johscheuer@users.noreply.github.com>
Joey Bloom <15joeybloom@users.noreply.github.com>
Jimmy Zhang <midnight.vivian@gmail.com>
Jeffrey Foster <jmfoste2@illinois.edu>
James Daniel <james-daniel@users.noreply.github.com>
Jake Bailey <zikaeroh@gmail.com>
JACKHAHA363 <luyuchen.paul@gmail.com>
Hydrosis <badda2k@gmail.com>
Hong <plantvsbird@gmail.com>
Grant Wu <grantwu2@gmail.com>
EvanFabry <Evan.Fabry@gmail.com>
EddieVilla <EddieVilla@users.noreply.github.com>
Deepak Nagaraj <n.deepak@gmail.com>
Daniel Meir Doron <danielmeirdoron@gmail.com>
Daniel Le <GreenRecycleBin@gmail.com>
Daniel Jamrozik <djamro2@illinois.edu>
Daniel Carballal <danielenriquecarballal@gmail.com>
Daniel <DTV96Calibre@users.noreply.github.com>
Daeyun Shin <daeyun@daeyunshin.com>
Creyslz <creyslz@gmail.com>
Christian Cygnus <gamer00@att.net>
CharlieMartell <charliecmartell@gmail.com>
Caleb Bassi <calebjbassi@gmail.com>
Brian Kurek <brkurek@gmail.com>
Brendan Wilson <brendan.x.wilson@gmail.com>
Bo Liu <boliu1@illinois.edu>
Ayush Ranjan <ayushr2@illinois.edu>
Atul kumar Agrawal <ms.atul1303@gmail.com>
Artur Sak <artursak1981@gmail.com>
Ankush Agarwal <ankushagarwal@users.noreply.github.com>
Angelino <angelino_m@outlook.com>
Andrey Zaytsev <andzaytsev@gmail.com>
Alex Yang <alyx.yang@gmail.com>
Alex Cusack <cusackalex@gmail.com>
Aidan Epstein <aidan@jmad.org>
Ace Nassri <ace.nassri@gmail.com>
Abdullahi Abdalla <abdalla6@illinois.edu>
Aneesh Durg <durg2@illinois.edu>
Assassin Eclipse <hungwoei96@hotmail.com>
Eric Cao <eric7252000@gmail.com>
Raphael Long <rafilong42@gmail.com>
williamsentosa95 <38774380+williamsentosa95@users.noreply.github.com>
Pradyumna Shome <pradyumna.shome@gmail.com>
Benjamin West Pollak <benjaminwpollak@gmail.com>
姜芃越 Pengyue Jiang <pengyue3@illinois.edu>
Andrew Orals <aorals2@illinois.edu>
Elijah Mock <emock3@illinois.edu>
Cay Zhang <13341339+Cay-Zhang@users.noreply.github.com>
```
