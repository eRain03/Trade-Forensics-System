<template>
    <h1>Hello Vue Router!</h1>
    <p>
        <strong>当前路由 path:</strong> {{ $route.fullPath }}
    </p>
<nav>
    <RouterLink to="/">首页</RouterLink>
    <RouterLink to="/about">关于</RouterLink>
</nav>
<main>
    <RouterView />
</main>
</template>