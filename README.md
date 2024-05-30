This text summarizes the back and forth over the P4TC patches[0]. 3 people (Alexei, Daniel and John) have nacked patch 14[9]. I am putting out this text to summarize the issues raised over a period and my responses because a lot of the details get lost in the weeds when the discussions get techno-emotional.
There have been many shifts in the views of the folks nacking, and despite my skepticism, my hope is if there was maybe a misunderstanding or miscommunication that this text will help clear things even for the folks disagreeing.

# Summary Of Our Requirements
To set context, I sill summarize our end goals. Over the last 18 months we have done many presentations and post lengthy commit logs, if you want to dig deeper start here: https://github.com/p4tc-dev/docs/blob/main/why-p4tc.md

## Why TC?

Our requirement is to implement P4 for both a s/w and h/w datapaths and the TC model which is widely deployed(and understood) fits in. Some reasons include:

 a) The P4 match-action paradigm has an excellent fit with TC match-action paradigm. There are other objects like P4 externs which also fit well with TC.

 b) Our target datapaths are both s/w and h/w - and the TC model already caters for this need.

 c) Netlink which provides a rich control interface that fits with P4 requirements for CRUD and event publish-subscribe semantics.

IOW, as noted above **TC meets our requirements** and avoids to have to invent new approaches for match-action tables or control CRUD or event publish-subscribe (and many others).

## P4TC Overview

High level P4TC architecure is illustrated in (**Figure 1**):
https://github.com/p4tc-dev/docs/blob/main/images/why-p4tc/p4tc-runtime-pipeline.png

The P4 objects that reside in the TC domain are shared between 1) s/w datapath 2) h/w datapath and 3) the control plane(in user space). The control plane uses netlink to target(see **Figure 1**) either h/w(<ins>flag=*skip_sw*</ins>) or s/w (<ins>flag=*skip_hw*</ins>) or both/best-effort-offload(<ins>flag=*0*</ins>).
If you are familiar with TC the above is no different from other classifier/actions that require offloads (eg *u32*, *flower*, etc).

See for example *flower* in (**Figure 2**): https://github.com/p4tc-dev/docs/blob/main/images/why-p4tc/tc-flower-offload.png

Notice in **Figure 1** illustrating P4TC: Either hardware or software datapath may be missing. eBPF is only used when we need a s/w datapath meaning that if one was only interested in h/w offload there would be no need to load the eBPF blobs. Likewise, as in all TC offloads, if your h/w doesnt support the equivalent h/w datapath then your driver will be lacking such support and reject any requests to offload; however, the <ins>same control path and object references are used for both h/w and s/w</ins>.

<!--
To provide extra comparison between P4TC (**Figure 1**) vs  other offload-capable classifiers like eg *matchall*, *u32* or *flower* (**Figure 2**): the *P4TC* s/w datapath is <ins>not hardcoded</ins>> in the kernel; in <ins>flower</ins> removing the s/w datapath(either by not compiling or unloading the *flower* kernel module) implies removing access to the h/w target. Other differences between P4TC and f.e *flower* is that in *P4TC* you could have <ins>infinite possible s/w and h/w datapaths</ins> whereas in *flower* the s/w datapath is hard-coded/fixed for both s/w and h/w. IOW, in P4TC you can describe a new datapath with new lookup approaches, new actions and datapath flow processing by writting a P4 program and then use the compiler to generate all the datapath and control glue (see the green color coding in **Figure 1**). Whereas in *flower* changing things even as trivial as a the lookup key requires not just changing the kernel s/w datapath for lookup and parsing, but also the driver, the control objects, netlink structure and user space code (iproute2, etc); and if you are to introduce a new action then you rinse and repeat that process. P4TC design intentionally avoids these code churns and brings a new way of thinking which highly reduces future kernel code maintanance.
-->

### What Are These Patches?

The posted series 1 patches[0] only introduce the <ins>s/w datapath as well as the necessary control objects</ins>. This is in conformance to the TC rule which states that: for TC datapaths which involve offloads it is required to introduce s/w twin before the h/w equivalent. The h/w offload patches are for a later series.

# The Pushback

Let me now provide a summary on the outstanding objections without going into the history to avoid relitigation (but as stated in the intro, the objection goal posts shifted many times); however, post typing this article in if newer discussions bring up some old arguement for a pushback then i will update this document (and add a date annotation with a link).

In the text below will provide my responses to each the raised issues. If i have missed or mis-stated any of these raised points please let me know and i will fix the text.

### 1a **Comment**: *"Use maps"*
This has been brought up numerous times from all 3 people and here's a sample:[1]

If you look at **Figure 1** above you will notice it <ins>doesnt make sense for our requirement</ins>.

1) eBPF is only in the picture for s/w datapath and can be removed while the h/w datapath continues to work fine. The control path, which accesses the objects, is for both h/w and s/w. If an operator wants a total offload then they dont need to load eBPF.
2)  P4 tables require a lot more features that are not natural for maps and P4 has a lot more objects than things that can be *squeezed* into maps.

For these reasons <ins>maps cannot be used</ins>.

#### 1b **Comment**: *"But there is already an eBPF P4 backend"*

Raised by John, so reusing the same reference [1]

Infact there is more than one eBPF backend for P4 written.  We use one of those backends as a basis for our implementation.

But: See **Figure 1** and then factor the sharing of objects the control path by s/w and h/w (and their optional presence).

<ins>None of the P4 eBPF outputs meet our requirement</ins>.

## 2a **Comment**: *"you should use bpf system calls for your control"*

Many references to this view from John and Daniel[1].

Again see **Figure 1** and observe netlink use, the object sharing involved, and then consider the absence of eBPF when only offload is needed.

But lets say "you want to make it work with bpf system call", it would mean having one control path for s/w and another for offload; it will also mean keeping two sets of P4 object objects, one set in TC and one in eBPF. Then of course reinvent everything netlink offers for the s/w datapath.

To put it crudely: <ins>Simply doesnt make sense</ins>.

### 2b **Comment**: *"but ... it is not performant"*

This has been brought up in regards to netlink and kfuncs by John and Daniel mostly but Alexei repeats it as well citing them.

My response has always consistently been: performance is a lower priority to P4 correctness and expressibility.

  1) Netlink provides us the control abstractions we need, it works with TC for both s/w and h/w offload and has a lot of knowledge base for expressing desired control plane APIs. We dont believe reinventing any of that makes sense.

  2) Kfuncs are a means to an end - they provide us the glue we need to have an optional ebpf s/w datapath get access to the TC objects. Getting an extra 10-100Kpps is **not** a motivating factor.

## 3 **Comment**: *"but you did it wrong, here's how you do it..."*

If you kept up with the exchanges you will notice that this sentiment is a big theme in the exchanges and consumed most of the electrons!  The problem here is a lot of these strong opinions are being expressed as "**expert opinions**" when in fact it is just basic bikesheding[11].

Some sample space:  A lot of sometimes downright insulting comments like [3], sometimes misleading statements like "use tcx"[2] or just a gut-feel opinions not based on any solid reasoning or experience like "disagree with pipeline design"[1]. Often when these statements were made we would invest time and investigate if we could somehow compromise and use the expressed ideas/opinions but often it turned out to be a wild goose chase and often if we  did "resolve" that "issue" another "do it this way" pops. See **Section #4c** for the a sample of such commentary...

Update May 28/2024: Here's another **vision** summarized as "Dont Do this in the kernel"[12] ;->

## 4 **Comments** which bring us to the current stalemate...

I will spend more text on this because it is still fresh in the most recent threads[3].

  a) **Comment**: *"...drop the kfunc patch 14 and i will drop my nack"*

  b) **Comment**: *"...all the object memory that is put in use by ebpf has to be freed when the ebpf program goes away"*

  c) **Paolo then trying to mediate** *"Can you enforce constraints such that all ebpf \"allocated memory\" is freed at program unload time"*

  #**Response 4a**

It is unreasonable to ask us to drop kfuncs because they are the main reason we migrated to eBPF[10] and invested 9 months re-writting the code at the strong insistence of eBPF collective with John and Daniel being quiet loud in that discussion.

kfuncs essentially boil down to function calls. They don't introduce new maintanance work to eBPF. IOW, no eBPF code is modified and any bug fixes/maintanance on the specific added calls falls on the P4TC side.
The going rules for kfuncs are: **they dont have to be sent to ebpf list or reviewed by folks in the ebpf world** - sample[4]. All our patches are in the TC domain and **dont touch any eBPF code**! Why is the rule different for P4TC? But lets assume for the sake of arguement that we had to involve the eBPF folks, then the question is: on what basis are they allowed to nack an invocation to a kernel function just because we used a specific way of invoking the function? Are kfuncs (or eBPF) infrastructure available to be used by the kernel community in general or not? I have no problem with technical review to impart coding rules etc - but is it ok to use kfuncs by other subsystems like TC for the sake of a feature? Lets have some consistent rules.

More importantly note that our implementation is literally **copied from existing kfuncs interacting with nf_conntrack**[5]! So it feels hypocritical that this rule does not apply to the conntrack code but applies to P4TC.

  #**Response 4b**
The conntrack code we copied from[5] **does not** free any memory allocated by kfunc used by the eBPF program. More importantly we would like to be in control of such policy decisions; example, one of the policies is keeping a table entry for a timeout period even after the eBPF program is gone (which is our current default behavior); or the eBPF program could be temporarily removed and a new version loaded using the same table (you dont want to be reloading a million entries at that point, etc). So dictating to us against our goal is frankly grasping for straws.

Update May 28/2024: Oh and here's another directly being approved by Alexei[14]

  #**Response 4c**
 I am only highlighting this part to demonstrate what i have called *shifting-goal-posts* and also our willingness to compromise throughout this ordeal. For example, as stated earlier, we spent 9 months of multiple people effort moving away from our original implementation in V1 which did not use eBPF to one using eBPF[10]. As mentioned earlier, John and Daniel had very strong opinions objecting to the V1 patches at the time and insisted strongly we need to use eBPF.

In response to Paolo's request, we spent a week analyzing our code and building a small prototype. **We then floated a compromise to make the change, not because we feel that the approach is correct(as explained in #4b) but in the hope that we can make progress**. IOW, we agreed to free any entries for tables that are created by the s/w datapath when the eBPF program is unloaded. But when Paolo asked if this approach would be good enough, the answer from Alexei was:
**"No. The whole design is broken. Remembering what was allocated by kfunc and freeing it later is not fixing the design at all."**[6].

Ok, so back in a cycle to **Comment #3**. We are doing it wrong and our design is broken ;->

## 5 Nobody Really wants This

Of course not true, we have had (still ongoing public) discussions and a long history (https://github.com/p4tc-dev/docs/blob/main/why-p4tc.md) involving multiple stake holders including vendors who support P4 abstractions in their NIC hardware. Unfortunately since we had to keep splaining this a few time it is worth to note it here as well. See the latest incarnation and see at least two very large vendors and someone very engaged in very large P4 projects explain why they need this[13a] [13b] [13c] - i think the other stake holders are confused on the kernel upstream politics and are tired of re-iterating their views (over and over and over).

## Now for a little rant

I was hoping to be wrong with that effort in **comment #4c** but having gone through so many of these cycles of a) making changes to appease mostly and then b) after posting patches, another excuse surfaces. Rinse, Repeat. At this point, I am extremely skeptical that there is good faith on the other side.

Open source is not a zero-sum game. EBPF already coexists with netfilter, tc, etc and various subsystems happily. I hope our requirements as stated above are clear and i dont have to keep justifying why P4 is needed or point to the vendors producing P4 native NICs or relitigate over and over again why we need TC. Open source is about scratching your itch and our itch is totally contained within TC i.e we only use eBPF as infra.

On my skeptic side, which i wish to be proven wrong, I cant help but feel that this community is getting way too pervasive with politics and obscure agendas. I understand agendas, I just dont understand the zero-sum thinking. These nacks are, intended or not, pushing off a whole community with an itch to scratch away from Linux kernel (yes, there's a whole industry behind P4). Paolo  says he cant apply these patches because 3 people(who are not network or tc maintainers) nacked them - in my opinion with no technical backing ;-> I am afraid, by definition this is abuse of power (and i dont mean Paolo here)! At minimal it signifies lack of set of rules to contain abuse of power; before corporatization of Linux such containment was based on good faith and meritocracy-driven good intentions.

If this nacks are to block these patches then we are in unprecedented cross-subsystem interference - like i stated earlier **zero changes are made in the eBPF domain**. Nobody in this community (i mean not even Linus) should have the power of saying "i am taking my ball[7] home because i dont like the way you are playing" without a reasonable and good technical backing. I dont believe there is a good technical backing in the nacks as i described above and so my skeptic view cant help but conclude there is no good faith on the other side and this needs to be checked[8].

# How do we move forward?

I am not asking to be given speacial treatment but it is clear we have a hole in the process currently. All i am asking for is fair treatment.
At this point, given that Paolo says the patches **cant be applied** because of 3 cross-subsystem <u>bogus</u> nacks, my suggestion on how we resolve this is for net maintainers to **appoint a third person arbitrator**. This person cannot be part of the TC or eBPF collective and has to be agreed to by both parties.

Hopefully this will introduce some new set of rules that will help the net maintainers resolve such issues should they surface in the future.

# References

[0]https://lore.kernel.org/all/20240410140141.495384-1-jhs@mojatatu.com/

[1]https://lore.kernel.org/all/661444789fccf_49a6208ec@john.notmuch/

[2]https://lore.kernel.org/netdev/CAM0EoMmm6SZawXy4wc=_LFKJFP6TFSKXQdCfRD4XrSON_AqDTA@mail.gmail.com/

[3]https://lore.kernel.org/netdev/CAADnVQLw1FRkvYJX0=6WMDoR7rQaWSVPnparErh4soDtKjc73w@mail.gmail.com/

[4]https://lore.kernel.org/bpf/CAADnVQ+RBV_rJx5LCtCiW-TWZ5DCOPz1V3ga_fc__RmL_6xgOg@mail.gmail.com/

[5]https://elixir.bootlin.com/linux/latest/source/net/netfilter/nf_conntrack_bpf.c#L318

[6]https://lore.kernel.org/all/CAADnVQLugkg+ahAapskRaE86=RnwpY8v=Nre8pn=sa4fTEoTyA@mail.gmail.com/

[7] proverbial ball being ebpf in this case. Literally the message is "you are not allowed to use ebpf" because i am the boss.

[8]Alexei made both suggestions and implicit threats that i push this to Linus directly with the nacks. See the exchange in [0].

[9]https://lore.kernel.org/all/20240410140141.495384-15-jhs@mojatatu.com/

[10] https://netdevconf.info/0x17/sessions/talk/integrating-ebpf-into-the-p4tc-datapath.html

[11] http://phk.freebsd.dk/sagas/bikeshed/

[12] https://lore.kernel.org/all/66566c7c6778d_52e720851@john.notmuch/

[13a] https://lore.kernel.org/all/CO1PR11MB499350FC06A5B87E4C770CCE93F42@CO1PR11MB4993.namprd11.prod.outlook.com/#t

[13b] https://lore.kernel.org/all/MW4PR12MB719209644426A0F5AE18D2E897F62@MW4PR12MB7192.namprd12.prod.outlook.com/

[13c] https://lore.kernel.org/all/SN6PR17MB21102057A1745DBCBB101DBB96F42@SN6PR17MB2110.namprd17.prod.outlook.com/

[14]  https://lore.kernel.org/all/CAADnVQJ_Wur5bmMsgOC7YvZ-D5GNzO9Fm2_4=L3eYkuQVpcg8g@mail.gmail.com/

