## Lotteon Tech Blog [블로그 바로가기](https://lotteon2.github.io)

## 기술 블로그 기여하기 
1. 기술 블로그 포스팅은 `_post` 폴더에 markdown 파일로 작성 
2. markdown 작성 시 헤더 작성 필수
```md
---
title: 포스팅 제목
description: 포스팅 설명
author: _data/authors.yml에 본인 내용 추가 후 사용 가능  
date: 2023-11-14 23:03 +0900 
lastmod: 2023-11-14 23:03 +0900
categories: msa
tags: kafka, msa, spring cloud
---
```
markdown 파일 헤더는 위와 같이 작성한다. 

3. `_data/authors.yml`에 내 정보 추가하기
```yaml
sangwon:
  name: Sang won
  url: https://github.com/nowgnas/
```
markdown 헤더에 `sangwon`으로 작성하면 된다. 

4. 글 내용 작성하기

markdown 문법에 따라 글을 작성하면 된다. 이미지, 표, 제목 등 간단한 문법만 사용해도 작성 가능하다. 

5. 포스팅 꿀팁

노션에 글을 작성하고 markdown으로 변환하게되면 파일 이름과 헤더만 추가하면 간편하게 포스팅 할 수 있다.    
이미지는 assets 폴더에 넣거나 github issue에 사진을 넣어 링크를 얻어 이미지를 넣으면 된다.    

6. 파일명 작성 방식

파일명은 yyyy-mm-dd-topic.md 로 작성한다. 
