# 志愿汇去广告 SimpleHook 配置

## SimpleHook

https://github.com/Xposed-Modules-Repo/me.simplehook

## 配置功能

去除开屏广告、弹窗广告、保险广告。

## 适用版本

在 5.4.4，5.5.8 测试通过，理论上长期可用。

## 配置

```json
{"packageName":"com.zzw.october","appName":"志愿汇","versionName":"5.5.8","description":"","configs":"[{\"className\":\"com.zzw.october.ad.MmAdManager\",\"methodName\":\"hasRequestAd\",\"resultValues\":\"true\",\"hookPoint\":\"after\",\"desc\":\"是否正在加载onStart广告，是那就不再加载了\"},{\"className\":\"com.zzw.october.LaunchActivity\",\"methodName\":\"isShowThirdAd\",\"resultValues\":\"false\",\"hookPoint\":\"after\",\"desc\":\"这样还会showOwnAd\"},{\"className\":\"com.zzw.october.bean.launch.StartUpBean$DataBean$OpenScreenAdvertiseBean\",\"methodName\":\"getCityAdvertSource\",\"resultValues\":\"0s\",\"hookPoint\":\"after\",\"desc\":\"showOwnAd里的判断条件，1为可展示\"},{\"className\":\"com.zzw.october.bean.InsuranceBean$DataBean\",\"methodName\":\"getStatus\",\"resultValues\":\"0\",\"hookPoint\":\"after\",\"desc\":\"志愿者保险弹窗/广告条，0为已购买\"}]","id":45}
```

