<template>
<form class="form login" method="post" @submit="submit">
    <h2 class="title"><img v-if="logo" alt="Sign in" :src="logo" /></h2>
    <div class="field">
        <div class="control">
            <input placeholder="Email" class="is-large input" type="email" v-model="email" />
        </div>
    </div>
    <div class="field">
        <div class="control">
            <input placeholder="Password" class="is-large input" type="password" v-model="pass" />
        </div>
    </div>
    <div class="field action">
        <div class="control">
            <button type="submit" :class="['button is-primary is-large is-fullwidth', {'is-loading': is_loading}]">Submit</button>
        </div>
        <p v-if="failed" class="help has-text-centered is-danger">Incorrect email or password!</p>
    </div>
</form>
</template>

<script>
export default
{
    name    :   'LoginForm',
    data() {
        return {
            email       :   null,
            pass        :   null,
            is_loading  :   false,
            logo        :   logo,
            failed      :   false
        }
    },
    watch   :   {
        email(nv, ov) {
            this.failed =   false;
        },
        pass(nv, ov) {
            this.failed =   false;
        }
    },
    methods :   {
        submit(e) {
            e.preventDefault();
            if (this.is_loading) return false;
            if (!this.email || this.email.trim().length == 0) return false;
            if (!this.pass || this.pass.trim().length == 0) return false;

            this.is_loading =   true;

            let me          =   this,
                params      =   new FormData();
            params.append('email', this.email);
            params.append('pass', this.pass);
            axios.post(
                base_url + endpoints.signin,
                params
            ).then((resp) => {
                me.$bus.$emit('onLive');
                me.is_loading   =   false;
            }).catch((error) => {
                me.is_loading   =   false;
                me.failed       =   true;
            });
        }
    }
}
</script>
