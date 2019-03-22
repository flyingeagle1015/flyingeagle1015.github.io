---
title: ThinkPHP3.2.3动态包含文件解决方案
date: 2017-06-23 11:36:46
tags: [php, thinkphp]
categories: [develop]
---

老手莫喷哦，我是PHP小菜！
使用场景：用户选择需要的页面，动态生成选项卡TAB页。
预想代码实现如下：
```php
<!-- 其他模板代码 -->
<div class="tabbable">
  <!-- nav-tabs 相关代码 -->
  <div class="tab-content">
    <foreach name="tabfiles" item="ff">
      <include file="$ff" />  <!--  ThinkPHP不支持，这里无法加载文件 -->
    </foreach>
  </div>
</div>
```

<!-- more -->

然而ThinkPHP3.2.3的include标签不支持动态解析。首先想到的是纯php实现文件内容读取，实现代码如下：
```php
<!-- 其他模板代码 -->
<div class="tabbable">
  <!-- nav-tabs 相关代码 -->
  <div class="tab-content">
    <foreach name="tabfiles_content" item="ff">  <!-- tabfiles_content是文件内容的数组 -->
      $ff          <!-- 直接输出文件内容 -->
    </foreach>
  </div>
</div>
```

另一种方法类似，只是把文件读取放在模板中，代码如下：
```php
<!-- 其他模板代码 -->
<div class="tabbable">
  <!-- nav-tabs 相关代码 -->
  <div class="tab-content">
    <foreach name="tabfiles" item="ff">
      <?php include_once($ff); ?> <!-- PHP 读取文件内容 -->
    </foreach>
  </div>
</div>
```

忍不住激动的心情，在浏览器里刷一下，页面真的出来了耶，然而细看一下发现PHP相关变量没有解析，显示也有问题~~~
经过查阅ThinkPHP的模板解析代码ThinkPHP/Library/Think/Template.class.php代码，终于感到了上天发现了它的秘密，
果断修改显示模板的实现：
```php
  //$this->display('mytabpage');修改为
  $this->show($this->fetch('mytabpage'));
```

哇哦，漂亮的页面终于出来了！！！
