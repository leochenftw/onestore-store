<template>
<form :class="['form store-form', {'come-down': input && input.trim().length}]" method="post" @submit="submit">
    <div class="field has-addons">
        <div class="control is-expanded">
            <div class="text is-fullwidth">
                <input autocomplete="off" ref="lookup_input" id="lookup" @keydown="keydown_handler" type="text" class="input is-small" v-model="input" />
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
            input       :   this.$route.query.receipt ? this.$route.query.receipt : null
        }
    },
    methods: {
        submit(e) {
            if (e) {
                e.preventDefault();
            }
            if (!this.input) return false;

            if (this.input == 'REFUND') {
                this.$parent.is_refunding   =   true;
                this.input  =   null;
                return;
            }

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

            if (this.$parent.is_refunding) {
                if (this.input.indexOf('RECEIPT-') > -1) {
                    this.input  =   null;
                    return false;
                }

                if (this.input.indexOf('DCNT-') > -1) {
                    this.$bus.$emit('showMessage', 'You cannot add discounts during refund process!', 'danger');
                    this.input  =   null;
                    return false;
                }
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
                } else if (resp.data.type == 'discount') {
                    me.$parent.toggle_discount(resp.data.data);
                } else {
                    me.$parent.reprint_receipt(resp.data.data);
                }
                me.$parent.toggle_loading();
            }).catch((error) => {
                me.is_loading   =   false;
                me.$parent.toggle_loading();
                if (error.response && error.response.data && error.response.data.message) {
                    me.$bus.$emit('showMessage', error.response.data.message);
                }
            });
        },
        keydown_handler(e)
        {
            if (e.keyCode == 27) {
                this.input    =   null;
            }
        },
        refocus(e)
        {
            if (can_query) {
                this.focusing();
            }
        },
        focusing()
        {
            $(this.$refs.lookup_input).focus();
        }
    },
    created() {
        let me  =   this;
        $(window).on('focus', function(e)
        {
            me.refocus();
        });

        if (this.$route.query && this.$route.query.receipt) {
            this.input  =   this.$route.query.receipt;
            this.submit();
        }
    },
    mounted() {
        can_query   =   true;
        let me      =   this;
        this.$nextTick().then(() => {
            this.refocus();
            $(this.$refs.lookup_input).on('blur', this.refocus);
        });
    }
}
</script>
