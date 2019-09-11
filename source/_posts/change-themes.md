---
title: 更换Hexo主题
date: 2019-09-11 13:38:30
type: search
tags: [themes,cactus,hexo]
---

### 原主题Next的Mist 

根目录下的`_config.yml`文件配置

```yml
themes: next
```

`themes/next`目录下`_config.yml`文件配置

```yaml
# Schemes
scheme: Mist
```

### 更为主题cactus

主题地址：[https://github.com/probberechts/hexo-theme-cactus](https://github.com/probberechts/hexo-theme-cactus)

#### 修改根目录下的`_config.yml`文件配置

```yaml
theme: cactus
theme_config:
  colorscheme: white  
```

#### 修改`themes/cactus`目录下`_config.yml`文件配置

```yaml
projects_url: http://github.com/akapril

nav:
  home: /
  # About: /about
  search: /search/

social_links:
  github: http://github.com/akapril
```

#### 配置搜索

安装`hexo-generator-search`插件

````shell
npm install hexo-generator-search --save
````

创建搜索页面

```shell
 hexo new page search
```

创建后内容修改成

```markdown
title: Search
type: search
---
```

最后在`themes/cactus`目录下`_config.yml`文件配置

```yaml
nav:
  search: /search/
```

`nav`下导航菜单可随意命名配置

```yaml
nav:
  主页: /
  搜索: /search/
```

