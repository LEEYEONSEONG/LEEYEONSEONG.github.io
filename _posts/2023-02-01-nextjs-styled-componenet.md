---
title: Next.js 에서의 styled-components 이슈
search: true
categories:
  - Next.js
---

server side rendering SSR 환경에서 styled components 를 사용해보자.

---

Next.js 프로젝트 초기에 서버 사이드 렌더링을 진행 하면서 한가지의 이슈를 발견했습니다.

- 웹 페이지 새로고침 시 css 파일이 뒤 늦게 읽혀 static 한 **_HTML 파일이 먼저 render가 되는 형상_**

로그아웃 시 새로고침이 이루어 지는데 로딩 컴포넌트틀 불러오는 과정에서 해당 컴포넌트의 HTML 파일이 잠깐 보이는 이슈가 생겼습니다.

Client-Side Rendering은 처음에 브라우저가 빈 HTML을 받아서 아무것도 보여주지 않다가, 사용자 기기에서 렌더링이 진행 되어 한번에 화면을 보여주게 되는데,

Next.js 는 기본적으로 페이지를 첫 로드 할때 SSR을 사용하여 Pre-rendering을 진행한다. 모든 일들을 클라이언트에서 수행하는것이 아니라 각 페이지의 HTML을 미리 생성하는 것이다.

# Pre-Rendering ?

### **initial Load**

자바스크립트 동작이 없는 static 한 HTML 파일을 화면에 먼저 보여준다.

### **hydration**

initial load 에서 html을 로드한 뒤 js 파일을 서버로 부터 받아 html을 연결시키는 과정을 갖는다

![styled component1](/assets/images/styled-component1.png "styled.img")

실제로 개발자 도구에서 javascript disabled를 진행하고 확인해 보면 같은 현상을 볼 수 있다.

Next.Js 에선 이에 대한 해결책을 제시 했는데, 공식 문서에서 제공해주는 [renderPage](https://nextjs.org/docs/advanced-features/custom-document)를 사용하여 html 파일에 `CSS-in-JS` 형식으로 작성된 스타일 요소들을 먼저 주입시켜 스타일이 뒤 늦게 적용 되는걸 방지해 줄 수 있다.

```ts
//pages/_document.tsx

import Document, {
  Html,
  Head,
  Main,
  NextScript,
  DocumentInitialProps,
  DocumentContext,
} from "next/document";
import { ServerStyleSheet } from "styled-components";

export default class MyDocument extends Document {
  static async getInitialProps(
    ctx: DocumentContext
  ): Promise<DocumentInitialProps> {
    const sheet = new ServerStyleSheet();
    const originalRenderPage = ctx.renderPage;

    try {
      ctx.renderPage = () =>
        originalRenderPage({
          enhanceApp: (App) => (props) =>
            sheet.collectStyles(<App {...props} />),
        });

      const initialProps = await Document.getInitialProps(ctx);
      return {
        ...initialProps,
        styles: [
          <>
            {initialProps.styles}
            {sheet.getStyleElement()}
          </>,
        ],
      };
    } finally {
      sheet.seal();
    }
  }

  render() {
    return (
      <Html>
        <Head>
          <meta charSet="utf-8"></meta>
          <meta
            content="upgrade-insecure-requests"
            httpEquiv="Content-Security-Policy"
          />
          <body>
            <Main />
            <NextScript />
          </body>
        </Head>
      </Html>
    );
  }
}
```

`_app.tsx` 파일은 next 프로젝트를 생성시에 만들어지게 되는데, `_document.tsx` 파일은 자동으로 만들어 지지 않는다.

페이지에 공통적으로 활용할 폰트 나 <body> 태그 안에 들어갈 내용들을 커스텀할때 활용한다. `_document.tsx` 는 서버에서 실행되고 `_app.tsx` 다음에 실행하게 된다.

styled components를 통해 페이지를 만들다 보면 페이지를 불러올 때 HTML 만 가져오고, 스타일이 적용되어있지 않다. 위 코드를 추가해줘야만 서버 사이드 렌더링 시 styled components가 헤더에 주입된다. 즉 서버에서 미리 HTML을 마크업할 때 스타일까지 HTML 요소에 녹여내는 것이다.

![styled2](/assets/images/styled-component2.png "trend.img")
