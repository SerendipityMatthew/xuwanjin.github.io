# Android Language

如何获取 android 系统的语言

对于大部分的的手机都是这么获取的
```kotlin
val locale: Locale = if (SDK_INT >= Build.VERSION_CODES.N) {
    resources.configuration.locales[0]
} else {
    resources.configuration.locale
}
val country = locale.country
val displayCountry = locale.displayCountry
val language = locale.language
val displayLanguage = locale.displayLanguage
val displayName = locale.displayName
Log.d(TAG, "onCreate: country = $country")
Log.d(TAG, "onCreate: displayCountry = $displayCountry")
Log.d(TAG, "onCreate: language = $language")
Log.d(TAG, "onCreate: displayLanguage = $displayLanguage")
Log.d(TAG, "onCreate: displayName = $displayName")

```
然而凡是都有例外,
对于 华为的 9.0 系统 (测试的手机是 Android 9.0, 华为 P10), 语言和地区是分开的
所以会出现 中文-巴西这样的奇葩情况
这个时候
``` kotlin
Settings.System.getString(application.contentResolver, "system_locales")
```
对于华为手机 会获的语言偏好的列表 比如在华为 android 9 手机上的列表是:
zh-Hant-BR,en-TW,zh-Hans-TW,ja-CN
排在第一位的语言就是, 系统的当前的语言, 中文-繁体-巴西. 所以现在的语言就是 Han traditional, 繁体字. 
虽然 android 10 禁止 hide 类型的接口, Settings.System.SYSTEM_LOCALES, 但是传入 相应的字符串
依然可以获得相应的返回值, 这个在 realme android 10 系统上已经测试过了.
或者


Java 语言的底层支持, 任何魔改的国产 Android 系统都一样
``` kotlin
Locale.getDefault().toLanguageTag()
```
在华为手机上测试的结果是: zh-Hant-BR

```kotlin
Locale.getDefault()
```
在华为手机上测试的结果是: zh_BR_#Hant

