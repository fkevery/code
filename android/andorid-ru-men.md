# Andorid 入门

## compileSdkVersion 与 targetSdkVersion

1. <mark style="color:red;">前者只影响编译时，不影响 app 真实运行情况</mark>。升级 compileSdkVersion 后有些新 api 可以用，但有些旧 api 可能会被标记为过时甚至被删除，所以会导致编译时出错。
2. <mark style="color:red;">后者影响 app 实际运行</mark>。有些 api 在不同的版本中会有所变化，因此升级 targetSdk 时可能会影响 app 实际运行效果。
