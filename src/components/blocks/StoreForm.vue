<template>
<form class="form store-form" method="post" @submit="submit">
    <div class="field has-addons">
        <div class="control is-expanded">
            <div class="text is-fullwidth">
                <input id="lookup" type="text" class="input is-small" v-model="input" />
            </div>
        </div>
        <div class="control">
            <button type="submit" :class="['button is-small is-primary', {'is-loading': is_loading}]">Lookup</button>
        </div>
    </div>
</form>
</template>

<script>
export default {
    name: 'StoreForm',
    data() {
        return {
            is_loading  :   false,
            input       :   null
        }
    },
    methods: {
        submit(e) {
            e.preventDefault();
            if (!this.input) return false;
            if (this.input == 'signout') {
                let me      =   this;
                axios.post(
                    base_url + endpoints.signout
                ).then((resp) => {
                    me.$bus.$emit('onLive');
                });

                this.input  =   null;
                return;
            }

            if (this.is_loading) return false;
            this.is_loading =   true;
            let me          =   this,
                params      =   new FormData();
            params.append('input', this.input);
            this.input      =   null;
            this.$parent.toggle_loading();
            axios.post(
                base_url + endpoints.lookup,
                params
            ).then((resp) => {
                me.is_loading   =   false;
                if (resp.data.type == 'product') {
                    me.$parent.add_item(resp.data.data);
                }
                me.$parent.toggle_loading();
            }).catch((error) => {
                me.is_loading   =   false;
                me.$parent.toggle_loading();
            });
        },
        focusing() {
            $(this.$el).find('.input').blur(function(e)
            {
                if (can_query) {
                    $(this).focus();
                }
            }).focus();
        }
    },
    created() {
        let me  =   true;
        $(window).on('focus', function(e)
        {
            if (can_query) {
                $('#lookup').focus();
            }
        });
    },
    mounted() {
        can_query   =   true;
        this.focusing();
    }
}
</script>
