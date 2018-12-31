<template>
<header id="header">
    <nav class="navbar is-transparent">
        <div class="navbar-brand">
            <router-link class="navbar-item" :to="{ name: 'Homepage' }">Home</router-link>
            <div v-on:click="show_mobile_menu" :class="{'navbar-burger': true, 'burger': true, 'is-active': mobile_menu_is_active}" data-target="mobile-menu">
                <span></span>
                <span></span>
                <span></span>
            </div>
        </div>
        <div id="mobile-menu" :class="{'navbar-menu': true, 'is-active': mobile_menu_is_active}">
            <div class="navbar-end">
                <div class="navbar-item" v-for="item in navigation">
                    <router-link :to="item.url">{{item.title}}</router-link>
                </div>
            </div>
        </div>
    </nav>
</header>
</template>
<script>
export default {
    name: 'Header',
    data: function() {
        return {
            mobile_menu_is_active: false,
            navigation: []
        }
    },
    methods: {
        click_to_close: function(e) {
            let target = $(e.target);
            if (!target.is('.burger') &&
                target.parents('.burger').length == 0 &&
                !target.is('.navbar-item') &&
                target.parents('.navbar-item').length == 0) {

                this.mobile_menu_is_active = false;
                $(window).unbind('mousedown', this.click_to_close);
            }
        },
        show_mobile_menu: function(e) {
            e.preventDefault();
            $(window).unbind('mousedown', this.click_to_close).on('mousedown', this.click_to_close);
            this.mobile_menu_is_active = !this.mobile_menu_is_active;
        }
    }
}
</script>
