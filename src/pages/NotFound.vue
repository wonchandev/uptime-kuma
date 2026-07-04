<template>
    <div class="not-found-wrap">
        <div class="not-found-card">
            <div class="code">404</div>
            <h1 class="headline">{{ $t("notFoundTitle") }}</h1>
            <p class="caption">{{ $t("notFoundDesc") }}</p>

            <div class="actions">
                <a href="/" class="btn-toss btn-toss-primary">{{ $t("notFoundGoHome") }}</a>
            </div>
        </div>
    </div>
</template>

<script>
export default {
    mounted() {
        document.body.classList.add("not-found-body");
    },
    unmounted() {
        document.body.classList.remove("not-found-body");
    },
};
</script>

<!-- eslint-disable-next-line vue-scoped-css/enforce-style-type -->
<style lang="scss">
// Page background (not scoped: applied on <body> while this page is mounted)
body.not-found-body {
    background-color: #f2f4f6;

    &.dark {
        background-color: #101014;
    }
}
</style>

<style scoped lang="scss">
@import "../assets/vars.scss";

.not-found-wrap {
    min-height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 24px;
}

.not-found-card {
    width: 100%;
    max-width: 440px;
    text-align: center;
    background-color: #fff;
    border-radius: 24px;
    padding: 40px 32px 36px;
    box-shadow: 0 1px 2px rgba(23, 32, 42, 0.04), 0 8px 24px rgba(23, 32, 42, 0.05);

    .dark & {
        background-color: #1e1e24;
        box-shadow: none;
    }
}

.code {
    display: inline-block;
    font-size: 15px;
    font-weight: 800;
    font-variant-numeric: tabular-nums;
    letter-spacing: 1px;
    padding: 6px 14px;
    border-radius: 50rem;
    background-color: rgba(1, 117, 49, 0.09);
    color: #017531;
    margin-bottom: 18px;

    .dark & {
        background-color: rgba(18, 163, 91, 0.16);
        color: #34c17e;
    }
}

.headline {
    font-size: 22px;
    font-weight: 800;
    letter-spacing: -0.4px;
    color: #191f28;
    margin-bottom: 8px;

    .dark & {
        color: #fff;
    }
}

.caption {
    font-size: 14px;
    line-height: 1.6;
    color: #8b95a1;
    margin-bottom: 28px;
}

.actions {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.btn-toss {
    display: block;
    width: 100%;
    border: none;
    border-radius: 14px;
    padding: 13px 16px;
    font-size: 15px;
    font-weight: 700;
    text-align: center;
    text-decoration: none;
    cursor: pointer;
    transition: filter 0.15s $easing-in;

    &:hover {
        filter: brightness(1.06);
        text-decoration: none;
    }
}

.btn-toss-primary {
    background-color: #017531;
    color: #fff;

    &:hover {
        color: #fff;
    }

    .dark & {
        background-color: #12a35b;
        color: #06130c;

        &:hover {
            color: #06130c;
        }
    }
}
</style>
