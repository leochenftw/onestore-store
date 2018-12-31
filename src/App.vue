<template>
<div id="app" :class="[class_name]">
    <main class="main">
        <router-view v-if="authenticated == true" />
        <LoginForm v-if="authenticated == false" />
    </main>
</div>
</template>

<script>
import Header from './components/blocks/Header';
import Footer from './components/blocks/Footer';
import slugify from 'slugify';
import LoginForm from './components/blocks/LoginForm';

export default {
    name: 'App',
    components: {
        Header,
        Footer,
        LoginForm
    },
    data() {
        return {
            authenticated   :   null
        }
    },
    computed: {
        class_name() {
            return slugify(this.$route.name, {lower: true});
        }
    },
    watch: {
        $route(to, from) {
            // this.$bus.$emit('onPageChange');
            // NProgress.start();
        }
    },
    created() {
        this.alive();
        this.$bus.$on('onLive', this.alive);
    },
    methods :   {
        alive() {
            let me                  =   this;
            me.authenticated        =   null;
            axios.get(
                base_url + endpoints.session
            ).then((resp) => {
                me.authenticated    =   true;
            }).catch((error) => {
                me.authenticated    =   false;
            });
        }
    }
}
</script>
<style>
/* @import '../node_modules/lightbox2/src/css/lightbox.css'; */
@import 'css/styles.css';
</style>
