## 0. 첫 NextJS 배포 
최근 회사에서 NextJS 프로젝트를 서버에 배포하는 임무를 맡아 진행하게 되었다. 스프링은 몇 번 배포해봤지만 Next JS 는 아직 배포해본적이 없어서 인터넷과 ai 를 적극 활용하였다. 스프링 어플리케이션 도커 배포와 크게 다를 건 없었지만 그래도 처음은 처음인 지라 (사실 NextJS 자체가 처음,,) 조금 시행 착오를 겪었다. 그래서 간만에 블로그 글도 채울겸 내가 겪었던 시행착오 및 새로 알게된 것들을 기록해보고자 한다. 

## 1. 첫번째 방황

```shell
FROM node:20-alpine

WORKDIR /app

# 의존성 파일 복사

COPY package.json package-lock.json* ./

# 의존성 설치
RUN npm ci

# 소스 코드 복사
COPY . .
  
# Next.js 애플리케이션 빌드
RUN npm run build
  
# 애플리케이션 실행을 위한 포트 노출
EXPOSE 3000

# 애플리케이션 실행
CMD ["npm", "start"]
```
사실 조금 급히 배포를 했어야하는 상황이라 인터넷을 최대한 빨리 검색하여 급히 Dockerfile 을 작성하였는데 그러다보니 도커이미지 용량이 너무 무거워지는 일이 발생했다. 

![[Pasted image 20250511161411.png]]

보다시피 1.34 GB 를 차지하는데 