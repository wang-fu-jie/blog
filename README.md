# blog

# 创建新博客
hugo new post/2024-11-18-my-first-blog.md

# 背景图库
https://pxhere.com/

# 本地调试
hugo server

# 更新索引
hugo

# 索引更新到云端
npm run algolia

# 博客提交
git add . && git commit -m 'DBA' && git push

# 手动给百度推资源
curl -H 'Content-Type:text/plain' --data-binary @urls.txt "http://data.zz.baidu.com/urls?site=https://www.wangfujie.cn&token=WDBX2AQwRbHeFVsj"