---
id: 160
title: android studio项目发布到jcenter 要点记录
date: 2015-12-16T17:07:08+00:00
author: roger
layout: post
views:
  - 43
duoshuo_thread_id:
  - 6228805651666043650
categories:
  - 所有文章
  - 技术
  - 首页
---
很多时候我们自己写了库，需要放到jcenter中以便快速的提供他人使用，以下记录了一些我在发布中遇到的问题，主要参考以下页面进行配置：
  
<a title="Rocko的博客" href="http://rocko.xyz/2015/02/02/%e4%bd%bf%e7%94%a8Gradle%e5%8f%91%e5%b8%83%e9%a1%b9%e7%9b%ae%e5%88%b0JCenter%e4%bb%93%e5%ba%93/" target="_blank">Rocko的博客</a>

下面是原文和我在操作时遇到的问题：
  
**申请Bintray账号(需要翻墙,如何翻请自行度娘~)**

Bintray的基本功能类似于Maven Central，一样的我们需要一个账号，<a title="传送门" href="https://bintray.com/" target="_blank">Bintray传送门</a>，注册完成后第一步算完成了。

生成项目的JavaDoc和source JARs

简单的说生成的这两样东西就是我们在下一步中上传到远程仓库JCenter上的文件了。这一步需要android-maven-plugin插件，所以我们需要在项目的build.gradle（Top-level build file，项目最外层的build.gradle文件）中添加这个构建依赖，如下：

<div class="codecolorer-container xml twitlight" style="overflow:auto;white-space:nowrap;width:100%;">
  <div class="xml codecolorer">
    buildscript {<br /> &nbsp; &nbsp; repositories {<br /> &nbsp; &nbsp; &nbsp; &nbsp; jcenter()<br /> &nbsp; &nbsp; }<br /> &nbsp; &nbsp; dependencies {<br /> &nbsp; &nbsp; &nbsp; &nbsp; classpath 'com.android.tools.build:gradle:1.0.0'<br /> &nbsp; &nbsp; &nbsp; &nbsp; classpath 'com.github.dcendents:android-maven-gradle-plugin:1.3'<br /> &nbsp; &nbsp; &nbsp; &nbsp; classpath 'com.jfrog.bintray.gradle:gradle-bintray-plugin:1.0'<br /> &nbsp; &nbsp; }<br /> }<br /> allprojects {<br /> &nbsp; &nbsp; repositories {<br /> &nbsp; &nbsp; &nbsp; &nbsp; jcenter()<br /> &nbsp; &nbsp; }<br /> }
  </div>
</div>

然后在你需要发布的那个module（我这里的即是library）的build.gradle里配置如下内容：**(只需要改有备注的地方)**

<!--more-->

<div class="codecolorer-container xml twitlight" style="overflow:auto;white-space:nowrap;width:100%;height:100%;">
  <div class="xml codecolorer">
    apply plugin: 'com.android.library'<br /> apply plugin: 'com.github.dcendents.android-maven'<br /> apply plugin: 'com.jfrog.bintray'<br /> // This is the library version used when deploying the artifact<br /> version = "1.0.0"<br /> android {<br /> &nbsp; &nbsp; compileSdkVersion 21<br /> &nbsp; &nbsp; buildToolsVersion "21.1.2"<br /> &nbsp; &nbsp; resourcePrefix "bounceprogressbar__" //这个随便填<br /> &nbsp; &nbsp; defaultConfig {<br /> &nbsp; &nbsp; &nbsp; &nbsp; minSdkVersion 9<br /> &nbsp; &nbsp; &nbsp; &nbsp; targetSdkVersion 21<br /> &nbsp; &nbsp; &nbsp; &nbsp; versionCode 1<br /> &nbsp; &nbsp; &nbsp; &nbsp; versionName version<br /> &nbsp; &nbsp; }<br /> &nbsp; &nbsp; buildTypes {<br /> &nbsp; &nbsp; &nbsp; &nbsp; release {<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; minifyEnabled false<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; proguardFiles getDefaultProguardFile('proguard-android.txt'),<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 'proguard-rules.pro'<br /> &nbsp; &nbsp; &nbsp; &nbsp; }<br /> &nbsp; &nbsp; }<br /> }<br /> dependencies {<br /> &nbsp; &nbsp; compile fileTree(dir: 'libs', include: ['*.jar'])<br /> &nbsp; &nbsp; compile 'com.nineoldandroids:library:2.4.0+'<br /> }<br /> def siteUrl = 'https://github.com/zhengxiaopeng/BounceProgressBar'<br /> // 项目的主页<br /> def gitUrl = 'https://github.com/zhengxiaopeng/BounceProgressBar.git'<br /> // Git仓库的url<br /> group = "org.rocko.bpb" // Maven Group ID for the artifact，一般填你唯一的包名<br /> install {<br /> &nbsp; &nbsp; repositories.mavenInstaller {<br /> &nbsp; &nbsp; &nbsp; &nbsp; // This generates POM.xml with proper parameters<br /> &nbsp; &nbsp; &nbsp; &nbsp; pom {<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; project {<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; packaging 'aar'<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; // Add your description here<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; name 'Android BounceProgressBar Widget' //项目描述<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; url siteUrl<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; // Set your license<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; licenses {<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; license {<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; name 'The Apache Software License, Version 2.0'<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; url 'http://www.apache.org/licenses/LICENSE-2.0.txt'<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; developers {<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; developer {<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; id 'zhengxiaopeng' //填写的一些基本信息<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; name 'Rocko'<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; email 'zhengxiaopeng.china@gmail.com'<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; scm {<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; connection gitUrl<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; developerConnection gitUrl<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; url siteUrl<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }<br /> &nbsp; &nbsp; &nbsp; &nbsp; }<br /> &nbsp; &nbsp; }<br /> }<br /> task sourcesJar(type: Jar) {<br /> &nbsp; &nbsp; from android.sourceSets.main.java.srcDirs<br /> &nbsp; &nbsp; classifier = 'sources'<br /> }<br /> task javadoc(type: Javadoc) {<br /> &nbsp; &nbsp; source = android.sourceSets.main.java.srcDirs<br /> &nbsp; &nbsp; classpath +=<br /> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; project.files(android.getBootClasspath().join(File.pathSeparator))<br /> }<br /> task javadocJar(type: Jar, dependsOn: javadoc) {<br /> &nbsp; &nbsp; classifier = 'javadoc'<br /> &nbsp; &nbsp; from javadoc.destinationDir<br /> }<br /> artifacts {<br /> &nbsp; &nbsp; archives javadocJar<br /> &nbsp; &nbsp; archives sourcesJar<br /> }<br /> Properties properties = new Properties()<br /> properties.load(<br /> &nbsp; &nbsp; &nbsp; &nbsp; project.rootProject.file('local.properties').newDataInputStream())<br /> bintray {<br /> &nbsp; &nbsp; user = properties.getProperty("bintray.user")<br /> &nbsp; &nbsp; key = properties.getProperty("bintray.apikey")<br /> &nbsp; &nbsp; configurations = ['archives']<br /> &nbsp; &nbsp; pkg {<br /> &nbsp; &nbsp; &nbsp; &nbsp; repo = "maven"<br /> &nbsp; &nbsp; &nbsp; &nbsp; name = "BounceProgressBar" //发布到JCenter上的项目名字<br /> &nbsp; &nbsp; &nbsp; &nbsp; websiteUrl = siteUrl<br /> &nbsp; &nbsp; &nbsp; &nbsp; vcsUrl = gitUrl<br /> &nbsp; &nbsp; &nbsp; &nbsp; licenses = ["Apache-2.0"]<br /> &nbsp; &nbsp; &nbsp; &nbsp; publish = true<br /> &nbsp; &nbsp; }<br /> }
  </div>
</div>

配置好上述后需要在你的项目的根目录上的local.properties文件里（一般这文件需gitignore，防止泄露账户信息）配置你的bintray账号信息，your\_user\_name为你的用户名，your_apikey为你的账户的apikey，可以点击进入你的账户信息里再点击Edit即有查看API Key的选项，把他复制下来。

<div class="codecolorer-container xml twitlight" style="overflow:auto;white-space:nowrap;width:100%;">
  <div class="xml codecolorer">
    bintray.user=your_user_name<br /> bintray.apikey=your_apikey
  </div>
</div>

Rebuild一下项目，顺利的话**(不顺利,这里会报一个错（Error:Cause: org/gradle/api/publication/maven/internal/DefaultMavenFactory Android）,<a title="解决方法" href="http://www.lai18.com/content/1417768.html" target="_blank">解决方法</a>)**，就可以在module里的build文件夹里生成相关文件了。这一步为止，就可以把你项目生成到本地的仓库中了，Android Studio中默认即在Android\sdk\extras\android\m2repository这里，所以我们可以通过如下命令(Windows中，可能还需要下载一遍Gradle，之后就不需要了)执行生成:

<div class="codecolorer-container xml twitlight" style="overflow:auto;white-space:nowrap;width:100%;">
  <div class="xml codecolorer">
    gradlew install
  </div>
</div>

上传到Bintray

上传到Bintray需要gradle-bintray-plugin的支持，所以在最外层的build.gradle里添加构建依赖：

<div class="codecolorer-container xml twitlight" style="overflow:auto;white-space:nowrap;width:100%;">
  <div class="xml codecolorer">
    buildscript {<br /> &nbsp; &nbsp; repositories {<br /> &nbsp; &nbsp; &nbsp; &nbsp; jcenter()<br /> &nbsp; &nbsp; }<br /> &nbsp; &nbsp; dependencies {<br /> &nbsp; &nbsp; &nbsp; &nbsp; classpath 'com.android.tools.build:gradle:1.0.0'<br /> &nbsp; &nbsp; &nbsp; &nbsp; classpath 'com.jfrog.bintray.gradle:gradle-bintray-plugin:1.0'<br /> &nbsp; &nbsp; &nbsp; &nbsp; classpath 'com.github.dcendents:android-maven-gradle-plugin:1.3'<br /> &nbsp; &nbsp; }<br /> }<br /> allprojects {<br /> &nbsp; &nbsp; repositories {<br /> &nbsp; &nbsp; &nbsp; &nbsp; jcenter()<br /> &nbsp; &nbsp; }<br /> }
  </div>
</div>

Rebuild一下，然后执行如下命令(Windows中)完成上传：

<div class="codecolorer-container xml twitlight" style="overflow:auto;white-space:nowrap;width:100%;">
  <div class="xml codecolorer">
    gradlew bintrayUpload
  </div>
</div>

**（我在这一步一直提示网络错误,后来加了调试信息gradlew bintrayUpload &#8211;stackover &#8211;debug 就成功鸟..不知为何）**
  
上传完成即可在Bintray网站上找到你的Repo，我们需要完成最后一步工作，申请你的Repo添加到JCenter。可以进入这个页面,输入你的项目名字点击匹配到的项目，然后写一写Comments再send即可，然后就等管理员批准了，我是大概等了40分钟，然后网站上会给你一条通过信息，然后就OK了，大功告成。**（我等了好几个小时 0 0）**

成功后就可以在其它项目里方便的使用你发布的项目了：

<div class="codecolorer-container xml twitlight" style="overflow:auto;white-space:nowrap;width:100%;">
  <div class="xml codecolorer">
    dependencies {<br /> &nbsp; &nbsp; compile 'org.rocko.bpb:library:1.0.0'<br /> }
  </div>
</div>

End