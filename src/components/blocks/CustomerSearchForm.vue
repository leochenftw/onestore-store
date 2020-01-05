<template>
<div id="customer-lookup-form" :class="{'is-active': is_active}">
    <form method="post" @submit.prevent="submit">
        <div class="field has-addons has-addons-centered">
            <p class="control">
                <input ref="phone_field" @keydown="keydown_handler" @blur="refocus" class="input" type="tel" v-model="phone" placeholder="02100000000">
            </p>
            <p class="control">
                <button type="submit" :class="['button is-danger', {'is-loading': is_loading}]">Lookup</button>
            </p>
        </div>
    </form>
    <div v-if="error_msg" class="column">
        <p class="help is-danger">{{error_msg}}</p>
    </div>
    <div v-else-if="customer" class="columns is-marginless customer-details">
        <div class="column">
            <p class="title is-6">Name: {{customer.first_name[0] + '*'}} {{customer.surname}}</p>
            <p class="subtitle is-7">Points: {{customer.shop_points}}</p>
        </div>
        <div class="column is-narrow">
            <button v-if="!$parent.customer" @click.prevent="set_customer" class="button is-info"><i class="fas fa-check"></i></button>
            <span v-else class="icon-confirmed"><i class="far fa-check-circle"></i></span>
        </div>
    </div>
</div>
</template>

<script>
export default {
    name    :   'CustomerSearchForm',
    props   :   ['is_active'],
    data()
    {
        return {
            is_loading  :   false,
            phone       :   location.hostname == 'localhost' ? '02108438078' : null,
            error_msg   :   null,
            customer    :   null,
            coupons     :   []
        }
    },
    watch   :   {
        is_active(nv, ov)
        {
            if (nv) {
                let me          =   this;
                this.customer   =   this.$parent.customer;
                this.coupons    =   this.$parent.coupons;
                this.$nextTick().then(() => {
                    setTimeout(() => {
                        $(me.$refs.phone_field).focus();
                    }, 100);
                });
            } else {
                this.customer   =   null;
                this.coupons    =   [];
                this.error_msg  =   null;
            }
        }
    },
    methods :   {
        refocus(e)
        {
            if (this.is_active) {
                $(this.$refs.phone_field).focus();
            }
        },
        keydown_handler(e)
        {
            if (e.keyCode == 27) {
                this.$parent.toggle_customer_lookup();
            }
        },
        set_customer(e)
        {
            if (this.customer) {
                this.$parent.customer   =   this.customer;
                this.$parent.coupons    =   this.coupons;
                this.$parent.toggle_customer_lookup();
            }
        },
        submit(e)
        {
            if (!this.phone && this.customer) {
                this.set_customer();
                return false;
            }

            if (this.is_loading) return false
            this.is_loading =   true;

            let me          =   this,
                params      =   new FormData();

            params.append('input', 'CUSTOMER-' + this.phone);
            this.phone      =   null;
            this.customer   =   null;
            this.coupons    =   [];
            this.error_msg  =   null;
            axios.post(
                base_url + endpoints.lookup,
                params
            ).then((resp) => {
                me.is_loading           =   false;
                this.customer           =   resp.data.customer;
                this.coupons            =   resp.data.available_coupons;
                this.$parent.coupons    =   [];
                this.$parent.customer   =   null;
            }).catch((error) => {
                me.is_loading   =   false;
                if (error.response && error.response.data && error.response.data.message) {
                    this.error_msg  =   error.response.data.message;
                }
            });
        }
    }
}
</script>
