## Abstract

Introducing how to Material UI 5 apply to Next.js 12 framework.

`SSR` Framework 에는 `Next.js`, `Nuxt`, `Sveltekit` 등 여러가지가 있지만 가장 대표적인 framework는 `react`기반의 `Next.js`일 것이다.

본 포스팅에서는 `Next.js`에 best react component library 중 하나인 `MUI`를 적용하는 가장 최신 방법을 소개하고자 한다.

기존의 `@mui/styles` 패키지를 통해 적용하는 방법은 `react 18`의 호환성 문제로 deprecated 되었으며 관련 내용은 하기 링크를 참조

The legacy styling solution of MUI: https://mui.com/system/styles/basics/

---

## Getting Started

<p>
  <img src="https://img.shields.io/badge/Next.js
-000000?style=flat-square&logo=Next.js
&logoColor=white"/>
  <img src="https://img.shields.io/badge/MUI-007FFF?style=flat-square&logo=MUI&logoColor=white"/>
  <img src="https://img.shields.io/badge/React-61DAFB?style=flat-square&logo=React&logoColor=white"/>
  <img src="https://img.shields.io/badge/TypeScript-3178c6?style=flat-square&logo=TypeScript&logoColor=white"/>
</p>

원하는 프로젝트 폴더에 `Next.js TypeScript` 프로젝트를 생성

##### Terminal

```sh
pnpm create next-app . --typescript
```

`MUI` 관련 패키지 설치

##### Terminal

```sh
pnpm add -S @emotion/cache @emotion/react @emotion/server @emotion/styled @mui/icons-material @mui/material
```

---

`createEmotionCache` 함수 모듈인 `emotionCache.ts` 를 `lib` 폴더에 생성

> #### Note
>
> - 클라이언트에서 `<head/>` 의 가장 최상단에 `<meta>` 태그를 생성하고 이를 `insertionPoint`로 지정한다.
>
> - 이는 페이지 로딩시 MUI style 을 가장 먼저 로딩하는것을 보장
>
> - 이렇게 먼저 로딩될 경우 다른 Style solution들 보다 높은 우선순위를 가지게 되어 `MUI`로 개발하는데 이점을 가지게 된다.

##### lib/emotionCache.ts

```ts
import createCache from '@emotion/cache';

const isBrowser = typeof document !== 'undefined';

const createEmotionCache = () => {
	let insertionPoint;

	if (isBrowser) {
		const emotionInsertionPoint = document.querySelector(
			'meta[name="emotion-insertion-point"]'
		) as HTMLElement;
		insertionPoint = emotionInsertionPoint ?? undefined;
	}

	return createCache({ key: 'mui-style', insertionPoint });
};

export default createEmotionCache;
```

---

`lib` 폴더에 `theme.ts` 도 생성

> #### Note
>
> - 나중에 global style을 적용하기 위해 theme.ts 생성

##### lib/theme.ts

```ts
import { createTheme } from '@mui/material/styles';
import { indigo } from '@mui/material/colors';

const theme = createTheme({
	palette: {
		primary: {
			main: indigo.A400,
		},
	},
});

export default theme;
```

---

`_app.tsx` 파일을 다음과 같이 수정

##### pages/\_app.tsx

```tsx
import * as React from 'react';
import Head from 'next/head';
import { ThemeProvider } from '@mui/material/styles';
import { CacheProvider } from '@emotion/react';
import CssBaseline from '@mui/material/CssBaseline';

import theme from '../lib/theme';
import createEmotionCache from '../lib/emotionCache';

import type { NextPage } from 'next';
import type { AppProps } from 'next/app';
import type { EmotionCache } from '@emotion/react';

type AppPropsWithCache = AppProps & {
	Component: NextPage;
	emotionCache?: EmotionCache;
};

const clientSideEmotionCache = createEmotionCache();

const MyApp = ({
	Component,
	emotionCache = clientSideEmotionCache,
	pageProps,
}: AppPropsWithCache) => {
	return (
		<CacheProvider value={emotionCache}>
			<Head>
				<meta name='viewport' content='initial-scale=1, width=device-width' />
			</Head>
			<ThemeProvider theme={theme}>
				<CssBaseline />
				<Component {...pageProps} />
			</ThemeProvider>
		</CacheProvider>
	);
};

export default MyApp;
```

---

`_document.tsx` 파일을 생성한 뒤 다음과 같이 작성

> #### Note
>
> - `_Document.getInitialProps`는 `_app`이 아닌 `_document`에 상속되며 static 으로 생성됨

```tsx
_Document.getInitialProps = async (ctx: DocumentContext): Promise<DocumentInitialProps> => {...}
```

- cache를 새로 생성하기보단 동일한 Emotion cache를 SSR Request에 사용한다면 퍼포먼스를 개선하는 효과를 얻을 수 있다. (하지만 global side effect가 발생할 수 있는 단점이 있다는 것도 인지해야한다.)

- 여기서는 SSR Request 마다 cache를 생성하기로 한다.

```tsx
const cache = createEmotionCache();
```

- `styles fragment`는 `app` 과 `page`의 렌더링이 끝난뒤에 렌더링된다.

```tsx
styles: [...React.Children.toArray(initialProps.styles), ...emotionStyleTags],
```

- 잘못된 HTML 생성 방지를 위해 다음 구문은 중요.

참고링크: https://github.com/mui/material-ui/issues/26561#issuecomment-855286153

```tsx
const emotionStyles = extractCriticalToChunks(initialProps.html);
```

> ### Generating Order
>
> #### 서버
>
> 1.  app.getInitialProps
>
> 2.  page.getInitialProps
>
> 3.  document.getInitialProps
>
> 4.  app.render
>
> 5.  page.render
>
> 6.  document.render
>
> #### 서버에러
>
> 1.  document.getInitialProps
>
> 2.  app.render
>
> 3.  page.render
>
> 4.  document.render
>
> #### 클라이언트
>
> 1.  app.getInitialProps
>
> 2.  page.getInitialProps
>
> 3.  app.render
>
> 4.  page.render

##### pages/\_document.tsx

```tsx
import * as React from 'react';
import Document, { Html, Head, Main, NextScript } from 'next/document';
import createEmotionServer from '@emotion/server/create-instance';
import theme from '../lib/theme';
import createEmotionCache from '../lib/emotionCache';

import type { DocumentContext, DocumentInitialProps } from 'next/document';

export default class _Document extends Document {
	render() {
		return (
			<Html lang='ko'>
				<Head>
					<meta name='theme-color' content={theme.palette.primary.main} />
					<link rel='shortcut icon' href='/static/favicon.ico' />
					<link
						rel='stylesheet'
						href='https://fonts.googleapis.com/css?family=Roboto:300,400,500,700&display=swap'
					/>
					<meta name='emotion-insertion-point' content='' />
					{this.props.styles}
				</Head>
				<body>
					<Main />
					<NextScript />
				</body>
			</Html>
		);
	}
}

_Document.getInitialProps = async (ctx: DocumentContext): Promise<DocumentInitialProps> => {
	const originalRenderPage = ctx.renderPage;

	const cache = createEmotionCache();
	const { extractCriticalToChunks } = createEmotionServer(cache);

	ctx.renderPage = () =>
		originalRenderPage({
			enhanceApp: (App) =>
				function EnhanceApp(props) {
					return <App emotionCache={cache} {...props} />;
				},
		});

	const initialProps = await Document.getInitialProps(ctx);

	const emotionStyles = extractCriticalToChunks(initialProps.html);
	const emotionStyleTags = emotionStyles.styles.map((style) => (
		<style
			data-emotion={`${style.key} ${style.ids.join(' ')}`}
			key={style.key}
			// eslint-disable-next-line react/no-danger
			dangerouslySetInnerHTML={{ __html: style.css }}
		/>
	));

	return {
		...initialProps,
		styles: [...React.Children.toArray(initialProps.styles), ...emotionStyleTags],
	};
};
```

---

`index.tsx` 를 다음과 같이 작성

###### pages/index.tsx

```tsx
import { Button } from '@mui/material';

import type { NextPage } from 'next';

const Home: NextPage = () => {
	return (
		<main
			style={{
				width: '100vw',
				height: '100vh',
				display: 'flex',
				justifyContent: 'center',
				alignItems: 'center',
			}}
		>
			<div>
				<header>
					<h1>MUI5 + Next.js 12</h1>
				</header>
				<section>
					<Button variant={'contained'}>Hello MUI</Button>
				</section>
			</div>
		</main>
	);
};

export default Home;
```

---

## Result

### 👉 [CodeSandBox Sample Link](https://codesandbox.io/p/github/soom-kang/Next.Js-12-MUI-5-Material-UI-New-Tutorial/main?workspaceId=99d1ebdb-6029-4fa4-945c-b71e7dfc3e5d&file=%2FREADME.md&workspace=%257B%2522activeFileId%2522%253A%2522clefgb6fp0001g4gfc5fucbcm%2522%252C%2522openFiles%2522%253A%255B%2522%252FREADME.md%2522%255D%252C%2522sidebarPanel%2522%253A%2522EXPLORER%2522%252C%2522gitSidebarPanel%2522%253A%2522COMMIT%2522%252C%2522spaces%2522%253A%257B%2522clefgb9ad00153b6l1h4fsjzw%2522%253A%257B%2522key%2522%253A%2522clefgb9ad00153b6l1h4fsjzw%2522%252C%2522name%2522%253A%2522Default%2522%252C%2522devtools%2522%253A%255B%257B%2522key%2522%253A%2522clefgb9ae00163b6lgz418wh6%2522%252C%2522type%2522%253A%2522PROJECT_SETUP%2522%252C%2522isMinimized%2522%253Atrue%257D%252C%257B%2522type%2522%253A%2522PREVIEW%2522%252C%2522taskId%2522%253A%2522dev%2522%252C%2522port%2522%253A3000%252C%2522key%2522%253A%2522clefgbqqq007y3b6lrlz5cak0%2522%252C%2522isMinimized%2522%253Afalse%257D%252C%257B%2522type%2522%253A%2522TASK_LOG%2522%252C%2522taskId%2522%253A%2522dev%2522%252C%2522key%2522%253A%2522clefgbpsj00583b6lxz9sr3mc%2522%252C%2522isMinimized%2522%253Afalse%257D%255D%257D%257D%252C%2522currentSpace%2522%253A%2522clefgb9ad00153b6l1h4fsjzw%2522%252C%2522spacesOrder%2522%253A%255B%2522clefgb9ad00153b6l1h4fsjzw%2522%255D%252C%2522hideCodeEditor%2522%253Afalse%257D)

![Result](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zti8h7trujgps8mwin0b.png)

---

## Conclusion

본 포스팅에서는 `MUI 5`를 `Next.Js 12`에 적용하는 최신 방법을 소개하였다. (2022년 8월 기준)

`react 18` 이 등장하면서 기존 방법보다 고려해야할 부분이 늘어난건 사실이나 여전히 reference가 많은 조합을 선호하는 트렌트 안에서는 Best 라고 할수있는 SSR Framework + UI Library 조합인 `Next.Js`+ `MUI`는 꾸준히 이용될 것이다.

이외에도 추천하는 React UI Framework 중 하나인 Mantine을 소개하면서 글을 마무리 짓고자 한다.
Mantine: https://mantine.dev/
