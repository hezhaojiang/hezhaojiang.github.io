---
title: 博客搭建 - NexT 主题美化
categories:
  - 博客搭建
tags:
  - Hexo
abbrlink: '97e2582'
date: 2020-10-11 14:23:25
---
{% note danger no-icon %}

看前必读：

本博客基于 `hexo v5.2.0` 和 `NexT v8.0.1` 搭建，本文的美化过程也是基于以上版本，不保证其他版本中的效果。

{% endnote %}

<!--more-->

## 增加背景图片

1. 修改主题配置文件（如 `_config.next.yml`）

    ``` linux
    # 找到以下内容
      # style: source/_data/styles.styl
    # 放开注释
      style: source/_data/styles.styl
    ```

2. 在 `source` 目录下 创建文件夹 `_data`，并在 `_data` 文件夹中创建名为 `styles.styl` 的新文件，并填入以下内容：

    ``` css
    // 增加背景图片
    body {
        background-image:url(http://www.dmoe.cc/random.php);
        background-repeat: no-repeat;
        background-attachment: fixed;
        background-position: center;
        //background-position:50% 50%;
        opacity: 0.95;
        +mobile(){
            background-image: url(https://acg.yanwz.cn/api.php);
            background-position: 10% 20%;
            background-size: cover;
        }
    }
    ```

    其中 `url` 可以使用网络图片地址，也可以使用本地图片，其他参数也可以根据 `css` 语法结合想要的效果自行修改。

## 页面元素美化

接增加背景图片，可通过在 `source/_data/styles.styl` 增加内容对页面元素进行美化

如增加元素阴影效果、修改元素圆角效果、优化文字间距等

具体修改如下：

``` css
// 调整组件位置
.posts-expand .post-header {
  font-size: 1.125em;
  margin-bottom: 10px;
  text-align: center;
}
.post-button {
  margin-top: 10px;
  text-align: center;
}

.post-block{
    margin-top: 10px;
    margin-bottom: 10px;
    padding: 10px;
}

// 组件边框椭圆化
.main-inner {
    background-color: rgba(255, 255, 255, 0);
    padding: 0px 10px 10px 10px;
    .sub-menu,
    .post-block,
    .tabs-comment,
    .comments,
    .pagination,
    .sub-menu:not(:first-child):not(.sub-menu),
    .post-block:not(:first-child):not(.sub-menu),
    .tabs-comment:not(:first-child):not(.sub-menu),
    .comments:not(:first-child):not(.sub-menu),
    .pagination:not(:first-child):not(.sub-menu) {
        border-radius: 30px 30px 30px 30px;
        box-shadow: 8px 7px 2px 0 rgba(0,0,0,0.12), 7px 4px 1px -2px rgba(0,0,0,0.06), 0 1px 5px 0 rgba(0,0,0,0.12);
    }
}

.header-inner {
    border-radius: 30px 30px 30px 30px;
    box-shadow: 8px 7px 2px 0 rgba(0,0,0,0.12), 7px 4px 1px -2px rgba(0,0,0,0.06), 0 1px 5px 0 rgba(0,0,0,0.12);
}

.sidebar-inner{
    border-radius: 30px 30px 30px 30px;
    box-shadow: 8px 7px 2px 0 rgba(0,0,0,0.12), 7px 4px 1px -2px rgba(0,0,0,0.06), 0 1px 5px 0 rgba(0,0,0,0.12);    
}

// 修改标题字体行距
h1,
h2,
h3,
h4,
h5,
h6 {
  font-family: Bree+Serif, "PingFang SC", "Microsoft YaHei", sans-serif;
  font-weight: bold;
  line-height: 1.5;
  margin: 0px 0 15px;
}

// 缩小打赏行距
.reward-container {
  margin: 5px;
  padding: 5px;
  text-align: center;
}
// 缩小相关文章行距
.popular-posts-header {
  border-bottom: 1px solid #eee;
  font-size: 1.125em;
  margin-bottom: 10px;
  margin-top: 10px;
}
// 缩小文章底部标签行距
.post-tags {
  margin-top: 10px;
  text-align: center;
}
```

## 增加网易云音乐

1. 修改 `layout/_partials/header/menu.njk` 文件，加入以下代码（也可以在自行在其他位置添加）：

    ``` js
    {%- if theme.background_music %}
        <div>
            <iframe seamless frameborder="no" border="0" marginwidth="0" marginheight="0" width=288 height=52 src="//music.163.com/outchain/player?type=2&id={{ theme.background_music }}&auto=1&height=32"></iframe>
        </div>
    {%- endif %}
    ```

    我添加的位置示例：

    ``` js
    {%- if theme.algolia_search.enable or theme.local_search.enable %}
        <li class="menu-item menu-item-search">
            <a role="button" class="popup-trigger">
            {%- if theme.menu_settings.icons %}<i class="fa fa-search fa-fw"></i>{%- endif %}{{ __('menu.search') }}
            </a>
        </li>
    {%- endif %}

    {%- if theme.background_music %}
        <div>
            <iframe seamless frameborder="no" border="0" marginwidth="0" marginheight="0" width=288 height=52 src="//music.163.com/outchain/player?type=2&id={{ theme.background_music }}&auto=1&height=32"></iframe>
        </div>
    {%- endif %}
    ```

2. 修改主题配置文件（如 `_config.next.yml`），添加如下内容：

    ``` linux
    background_music: 569214252
    ```

    > `background_music` 字段后的音乐 `id` 可用下图方式获取：
    > ![网易云音乐 id 获取方式](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201011225105.png)

## 增加点击特效

1. 创建 `source/js/click_magic.js` 文件，填入以下内容：

    ``` js
    /* 鼠标特效 */
    var a_idx = 0;
    jQuery(document).ready(function($) {
    $("body").click(function(e) {
        var a = new Array("Python", "Java", "Go", "C", "C++", "C#", "JavaScript" ,
        "Php", "Sql", "R");
        var $i = $("<span/>").text(a[a_idx]);
        a_idx = (a_idx + 1) % a.length;
        var x = e.pageX,
        y = e.pageY;
        $i.css({
            "z-index": 9999,
            "top": y - 20,
            "left": x,
            "position": "absolute",
            "font-weight": "bold",
            "color": "#ff6651"
        });
        $("body").append($i);
        $i.animate({
            "top": y - 180,
            "opacity": 0
        },
        1500,
        function() {
            $i.remove();
        });
    });
    });
    ```

2. 修改 `layout/_layout.njk`，在该文件最末尾增加如下代码：

    ``` js
    <script type="text/javascript" src="/js/click_magic.js"></script>
    ```

## 增加友链链接

1. 创建 `layout/_partials/page/links.njk` 文件，填入以下内容：

    ``` js
    {% block content %}
    {######################}
    {### LINKS BLOCK ###}
    {######################}

        <div id="links">
            <style>
                #links{
                margin-top: 5rem;
                }
                .links-content{
                    margin-top:1rem;
                }
                .link-navigation::after {
                    content: " ";
                    display: block;
                    clear: both;
                }
                .card {
                    width: 300px;
                    font-size: 1rem;
                    padding: 10px 20px;
                    border-radius: 4px;
                    transition-duration: 0.15s;
                    margin-bottom: 1rem;
                    display:flex;
                }
                .card:nth-child(odd) {
                    float: left;
                }
                .card:nth-child(even) {
                    float: right;
                }
                .card:hover {
                    transform: scale(1.1);
                    box-shadow: 0 2px 6px 0 rgba(0, 0, 0, 0.12), 0 0 6px 0 rgba(0, 0, 0, 0.04);
                }
                .card a {
                    border:none;
                }
                .card .ava {
                    width: 3rem!important;
                    height: 3rem!important;
                    margin:0!important;
                    margin-right: 1em!important;
                    border-radius:4px;
                }
                .card .card-header {
                    font-style: italic;
                    overflow: hidden;
                    width: 236px;
                }
                .card .card-header a {
                    font-style: normal;
                    color: #2bbc8a;
                    font-weight: bold;
                    text-decoration: none;
                }
                .card .card-header a:hover {
                    color: #d480aa;
                    text-decoration: none;
                }
                .card .card-header .info {
                    font-style:normal;
                    color:#a3a3a3;
                    font-size:14px;
                    min-width: 0;
                    text-overflow: ellipsis;
                    overflow: hidden;
                    white-space: nowrap;
                }
                span.focus-links {
                    font-style: normal;
                    margin-left: 10px;
                    position: unset;
                    left: 0;
                    padding: 0 7px 0 5px;
                    font-size: 11px;
                    border-color: #42c02e;
                    border-radius: 40px;
                    line-height: 24px;
                    height: 22px;
                    color: #fff !important;
                    background-color: #42c02e;
                    display: inline-block;
                }
                span.focus-links:hover{
                    background-color: #318024;
                }
            </style>
            <div class="links-content">
                <div class="no-icon note warning"><div class="link-info">👨‍🎓 跟着大佬走，成为小大佬</div></div>
                <div class="link-navigation">
                    {% for link in theme.defaultlinks %}
                        <div class="card">
                            <img class="ava nofancybox" src="{{ link.avatar }}"/>
                            <div class="card-header">
                            <div><a href="{{ link.site }}" target="_blank"> {{ link.nickname }}</a> <a href="{{ link.site }}"><span class="focus-links"><i class="fa fa-plus" aria-hidden="true"></i>&nbsp;关注</span></a></div>
                            <div class="info">{{ link.info }}</div>
                            </div>
                        </div>
                    {% endfor %}
                </div>
                <div class="no-icon note primary"><div class="link-info">🍭 五湖四海的朋友们</div></div>
                <div class="link-navigation">
                    {% for link in theme.friendslinks %}
                        <div class="card">
                            <img class="ava nofancybox" src="{{ link.avatar }}"/>
                            <div class="card-header">
                            <div><a href="{{ link.site }}" target="_blank"> {{ link.nickname }}</a> <a href="{{ link.site }}"><span class="focus-links"><i class="fa fa-plus" aria-hidden="true"></i>&nbsp;关注</span></a></div>
                            <div class="info">{{ link.info }}</div>
                            </div>
                        </div>
                    {% endfor %}
                </div>
                {{ page.content }}
                </div>
            </div>

    {##########################}
    {### END LINKS BLOCK ###}
    {##########################}
    {% endblock %}
    ```

2. 修改 `layout/page.njk`：

    ``` js
    /* 找到如下两行代码 */
    {%- elif page.type === 'schedule' and not page.title %}
        {{- __('title.schedule') + page_title_suffix }}
    /* 在上述代码后添加如下内容 */
    {%- elif page.type === 'links' and not page.title %}
        {{- __('title.links') + page_title_suffix }}

    /* 找到如下两行代码 */
    {% elif page.type === 'schedule' %}
        {% include '_partials/page/schedule.njk' %}
    /* 在上述代码后添加如下内容 */
    {% elif page.type === 'links' %}
        {% include '_partials/page/links.njk' %}
    ```

3. 修改主题配置文件（如 `_config.next.yml`）：

    ``` linux
    # 找到如下代码
    archives: /archives/ || fa fa-archive
    # 在上述代码后添加如下内容
    links: /links || fa fa-link
    ```

4. 修改语言文件 `languages/zh-CN.yml`:

    ``` linux
    # 在合适位置添加以下内容
    links: 友链
    # 比如
    about: 关于
    links: 友链
    ```

5. 修改主题配置文件（如 `_config.next.yml`），添加如下内容：

    ``` linux
    # defaultlinks 会显示在 跟着大佬走，成为小大佬 标签中
    defaultlinks:
      - nickname: 雷霄骅
        site: https://me.csdn.net/leixiaohua1020
        avatar: https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201008135337.png
        info: 中国传媒大学一个搞广播电视相关的视音频技术的学生

    # friendslinks 会显示在 五湖四海的朋友们 标签中
    friendslinks:
      - nickname: 何照江
        site: https://xiaoqingdu.gitee.io/
        avatar: https://xiaoqingdu.gitee.io/images/head.jpeg
        info: 无
    ```
