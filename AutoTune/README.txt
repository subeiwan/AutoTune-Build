AutoTune 1.0（源码，未编译）
============================

这是猎枪弹道数据采集模块的 Theos 工程源码。**这里没有编译好的 dylib**，
原因后面第四段写清楚了，别急着骂，看完就知道怎么拿到 dylib。


一、它干什么
------------

挂在游戏进程里，按 50 米一段统计射击情况（跟游戏面板的分段一致），
结果写到 App 的 Documents/autotune.json，导出来丢进 ZakaSign 的「对局数据」页，
就出分段命中率表和「哪段该加多少下坠补偿」的建议。

输出格式（20 段以内，打过的段才写）：

  {
    "version": 1,
    "segment": 50,
    "segments": [
      {"index": 2, "shots": 16, "hits": 4, "err_v": -3.2, "err_h": -0.9, "err_n": 16}
    ]
  }

  index   = 距离段序号（2 = 100~150 米）
  err_v   = 平均纵向偏差（负 = 落点在目标后面，该加下坠补偿）
  err_h   = 平均横向偏差


二、怎么编译
------------

有三条路，随便挑一条：

A. 有 Mac
   brew install ldid  （theos 依赖）
   git clone --recursive https://github.com/theos/theos.git ~/theos
   下载 iOS SDK 放进 ~/theos/sdks/
   cd AutoTune && make package FINALPACKAGE=1
   产物在 .theos/obj/ 里，就是 .dylib

B. 没 Mac，用 GitHub Actions（推荐，免费）
   把这个 AutoTune 目录推到你自己的 GitHub 仓库，
   Actions 里有现成的 build-autotune 工作流，点一下就跑，跑完在 Artifacts 里下 dylib。

C. 有 Windows + WSL
   装 WSL2 → 在 WSL 里按 A 的步骤装 Theos（不用 Mac），能编出来。


三、编出来怎么用
----------------

1. 打开 ZakaSign →「签名」页
2. IPA 选游戏包，p12 + 描述文件照常给
3.「注入 dylib」这一栏把编出来的 AutoTune.dylib 选上
4. 开始签名 → 装到手机

装好后第一次进游戏，去 App 的 Documents 里就能看到 autotune.json（说明模块活着）。


四、为什么我没直接给你 dylib
----------------------------

我这边的工作环境里没有 iOS 工具链（没有 clang 的 iOS target、没有 SDK、没有 Theos），
编不出能被 iOS 加载的 arm64 dylib。硬给你一个假的那是害你——装上去要么闪退要么白屏。

所以给你的是完整源码 + 三条能出 dylib 的路（其中 GitHub Actions 那条不需要 Mac，十分钟搞定）。
你编译出来之后，签名、注入、装机这些我这边全都接得住。


五、要采到真实数据，还差最后一步
--------------------------------

源码装钩子的地方是 config 驱动的，现在 offset 是 0x0，不填就只写一个空文件（能证明模块加载成功）。
要真采数据，得先知道游戏里射线判定函数的偏移：

1. 用 il2cpp dump / IDA 找到 UnityFramework 里的 Physics.Raycast
2. 把镜像名和偏移填进 Documents/autotune_hook.json：

  {
    "hooks": [
      {"image": "UnityFramework", "offset": "0x1A2B3C4", "kind": "raycast"}
    ]
  }

3. 重启游戏，打几枪，autotune.json 里就有分段数据了

猎枪是实体弹丸，射线只能告诉你"瞄的方向对不对"，弹丸真实落点还要再往下挖一层
（找弹丸类的速度/位置更新函数，把 kind 换成对应的处理函数）。
第一批数据我建议先采射线，把「哪些距离段明显打不准」画出来，再决定要不要深挖弹丸。

把 autotune.json 和 dump 出来的类名清单发我，下一版我直接对接。
