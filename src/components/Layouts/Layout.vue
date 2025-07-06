<template>
    <header>
        <div>
            <img src="../../assets/favicon.svg">
            <span class="text-gradient">Songs Classifier</span>
        </div>
    </header>
    <main>
        <slot />
    </main>
    <footer>
        <small>Created by</small>
        <a href="https://github.com/kevinmahrous" target="_blank">
            <p>kevinmahrous</p>
        </a>
    </footer>
</template>

<style scoped>
header,
footer {
    align-items: center;
    padding: 1rem;
    width: 100%;
    max-width: 600px;
}

header div span {
    font-size: 1.5rem;
    font-weight: 600;
}

footer {
    margin: 0 auto;
}

header div {
    display: flex;
    align-items: center;
    gap: 0.75rem;
    padding-left: 0.5rem;
}

header img {
    width: 2rem;
    height: 2rem;
}

footer {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    align-items: center;
    padding: 2rem 0;
    padding-bottom: 3rem;
}

footer a {
    display: flex;
    align-items: center;
    padding: 0.5rem;
    padding-right: 0.5rem;
    background: var(--background-muted);
    border-radius: 4rem;
    border: 1px solid transparent;
    transition-duration: 200ms;
    text-decoration: none;
}

footer a:hover {
    border-color: var(--color-link);
}
</style>