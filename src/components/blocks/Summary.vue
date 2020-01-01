<template>
<div :class="['container summary', extra_classes]">
    <div class="columns is-mobile">
        <div class="column" v-if="!receipt">
            <template v-if="!is_free">
                <button :disabled="cash_disabled" @click.prevent="give_change" :class="['is-outlined button is-large is-warning', {'is-active': payment_method == 'Cash'}]">CASH</button>
                <button :disabled="eftpos_disabled" @click.prevent="wait_eftpos" :class="['is-outlined button is-large is-primary', {'is-active': payment_method == 'EFTPOS'}]">EFTPOS</button>
            </template>
            <button v-else :disabled="voucher_disabled" @click.prevent="voucher_payment" :class="['is-outlined button is-large is-danger', {'is-active': payment_method == 'Voucher'}]">VOUCHER</button>
        </div>
        <div class="column" v-else>
            <button class="button is-large is-danger" @click.prevent="do_exit">Exit</button>
            <button class="button is-large is-info is-outlined" @click.prevent="do_print">Print</button>
            <button v-if="payment_method == 'Cash'" class="is-outlined button is-large is-warning is-active">CASH</button>
            <button v-if="payment_method == 'EFTPOS'" class="is-outlined button is-large is-primary is-active">EFTPOS</button>
            <button v-if="payment_method == 'Voucher'" class="is-outlined button is-large is-danger is-active">VOUCHER</button>
        </div>
        <div class="column has-text-right">
            <p class="title is-1 discount-total">
                <button @click.prevent="$parent.cancel_discount" class="is-small is-danger button" v-if="discount">{{discount.title}}</button>
                <span class="is-tiny" v-if="discount">-{{discount.by == '%' ? (discount.rate + '%') : (discount.rate.toDollar())}}</span>
                <button v-if="$parent.coupons.length" @click.prevent="check_coupons" :class="['hide-in-print button is-danger is-circle', {'bouncy': !coupons_clicked}]"><span class="icon"><i v-if="!$parent.coupons_shown" class="fas fa-gift"></i><i class="fas fa-times" v-else></i></span></button>
                {{total_amount.toDollar()}}
            </p>
            <p class="subtitle is-7">
                incl. GST {{gst}}
                <template v-if="cash_taken"><br />Received {{cash_taken.toDollar()}} cash; Given {{given_change}} change</template>
            </p>
        </div>
    </div>
</div>
</template>

<script>
export default {
    name        :   'Summary',
    props       :   ['total', 'discount', 'extra_classes', 'receipt'],
    data() {
        return {
            coupons_clicked     :   false,
            cash_loading        :   false,
            eftpos_loading      :   false,
            voucher_loading     :   false,
            cash_disabled       :   false,
            eftpos_disabled     :   false,
            voucher_disabled    :   false,
            payment_method      :   null,
            cash_taken          :   null
        }
    },
    watch       :   {
        discount(nv, ov)
        {
            if (nv) {

            } else {

            }
        },
        receipt(nv, ov) {
            if (nv) {
                this.cash_taken =   this.receipt.cash;
            }
        }
    },
    created() {
        let me  =   this;
        this.$bus.$on('onMethdDominated', (method) => {
            me.payment_method   =   method;
        });
    },
    beforeDestroy() {
        this.$bus.$off('onMethdDominated');
    },
    computed    :   {
        given_change() {
            let amount  =   this.cash_taken ? this.cash_taken.toFloat() : 0,
                change  =   amount - (Math.round(this.total * 10) * 0.1);
            change      =   change > 0 ? change : 0;
            return change.toDollar();
        },
        is_free()
        {
            if (this.total && this.$parent.coupon) {
                let total   =   this.total;

                if (this.discount) {
                    if (this.discount.by == '%') {
                        total   =   total * (1 - this.discount.rate * 0.01);
                    } else {
                        total   =   total - this.discount.rate;
                    }
                }

                return total <= 0;
            }

            return false;
        },
        total_amount() {
            if (this.total) {
                let sum =   this.total;
                if (this.discount) {
                    if (this.discount.by == '%') {
                        sum     =   (sum * (1 - this.discount.rate * 0.01));
                        sum     =   sum < 0 ? 0 : sum;
                    } else {
                        sum     =   (sum - this.discount.rate);
                        sum     =   sum < 0 ? 0 : sum;
                    }

                    return sum;
                }

                return this.total;
            }

            return 0;
        },
        gst() {
            return (this.total_amount * 0.15 / 1.15).toDollar();
        }
    },
    methods     :   {
        check_coupons()
        {
            this.coupons_clicked    =   true;
            this.$parent.toggle_coupons();
        },
        do_exit() {
            let me  =   this;
            me.cash_loading     =   false;
            me.eftpos_loading   =   false;
            me.voucher_loading  =   false;
            me.cash_disabled    =   false;
            me.eftpos_disabled  =   false;
            me.voucher_disabled =   false;
            me.payment_method   =   null;
            me.$router.replace('/');
            me.$parent.reset();
        },
        do_print() {
            window.print();
            this.slient_reset();
        },
        give_change() {
            this.payment_method =   'Cash';
            this.$bus.$emit('giveChange', this.total_amount, this.place_order);
        },
        wait_eftpos() {
            this.payment_method =   'EFTPOS';
            this.$bus.$emit('waitEFTPOS', this.total_amount, this.place_order);
        },
        voucher_payment()
        {
            this.place_order('Voucher');
        },
        place_order(by, amount_taken) {
            if (this.cash_loading || this.eftpos_loading || this.voucher_loading) return false;

            if (by == 'EFTPOS') {
                this.eftpos_loading     =   true;
                this.cash_disabled      =   true;
                this.voucher_disabled   =   true;
            } else if (by == 'Cash'){
                this.cash_loading       =   true;
                this.eftpos_disabled    =   true;
                this.voucher_disabled   =   true;
            } else {
                this.voucher_loading    =   true;
                this.cash_disabled      =   true;
                this.eftpos_disabled    =   true;
            }

            let me      =   this,
                params  =   new FormData();

            params.append('by', by);
            params.append('list', JSON.stringify(this.$parent.goods));
            if (amount_taken) {
                this.cash_taken =   amount_taken;
                params.append('cash_taken', this.cash_taken);
            }

            if (this.$parent.customer) {
                params.append('customer', this.$parent.customer.id);
                if (this.$parent.coupon) {
                    params.append('coupon', this.$parent.coupon.id);
                }
            }

            if (this.discount && !this.$parent.coupon) {
                params.append('discount', this.discount.id);
            }

            axios.post(
                base_url + endpoints.order,
                params
            ).then((resp) => {
                me.$parent.do_receipt(resp.data);
                me.$nextTick().then(() => {
                    me.$nextTick().then(() => {
                        window.print();
                        me.reset();
                    });
                });
            }).catch((error) => {
                me.cash_loading     =   false;
                me.eftpos_loading   =   false;
                me.cash_disabled    =   false;
                me.eftpos_disabled  =   false;
                me.voucher_disabled =   false;
                me.payment_method   =   null;
                me.cash_taken       =   null;
                if (error.response && error.response.data && error.response.data.message) {
                    me.$bus.$emit('showMessage', error.response.data.message);
                }
            });
        },
        slient_reset() {
            let me  =   this;
            me.cash_loading     =   false;
            me.eftpos_loading   =   false;
            me.cash_disabled    =   false;
            me.eftpos_disabled  =   false;
            me.voucher_disabled =   false;
            me.payment_method   =   null;
            me.cash_taken       =   null;
            me.coupons_clicked  =   false;
            if (me.$route.name != 'Homepage') {
                me.$router.replace('/');
            }
            me.$parent.reset();
        },
        reset() {
            let me  =   this;
            me.$bus.$emit('showMessage', 'Thank you!', 'success', 'Completed', () => {
                me.cash_loading     =   false;
                me.eftpos_loading   =   false;
                me.cash_disabled    =   false;
                me.eftpos_disabled  =   false;
                me.voucher_disabled =   false;
                me.payment_method   =   null;
                me.cash_taken       =   null;
                me.coupons_clicked  =   false;
                if (me.$route.name != 'Homepage') {
                    me.$router.replace('/');
                }
                me.$parent.reset();
            });
        }
    }
}
</script>
