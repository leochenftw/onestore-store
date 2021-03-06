<template>
<div :class="['container summary', extra_classes]">
    <div class="columns is-mobile">
        <div class="column" v-if="!receipt">
            <template v-if="!is_free">
                <button :disabled="cash_disabled" @click.prevent="give_change" :class="['is-outlined button is-large is-warning', {'is-active': payment_method == 'Cash'}]">CASH</button>
                <button :disabled="eftpos_disabled" @click.prevent="wait_eftpos" :class="['is-outlined button is-large is-primary', {'is-active': payment_method == 'EFTPOS'}]">EFTPOS</button>
                <button :disabled="web_disabled" @click.prevent="wait_web" :class="['is-outlined button is-large is-info', {'is-active': payment_method == 'Web Order'}]">Web Order</button>
            </template>
            <button v-else :disabled="voucher_disabled" @click.prevent="voucher_payment" :class="['is-outlined button is-large is-danger', {'is-active': payment_method == 'Voucher'}]">VOUCHER</button>
        </div>
        <div :class="['column', {'align-centered' : $parent.is_refunding}]" v-else>
            <template v-if="!$parent.is_refunding">
                <button class="button is-large is-danger" @click.prevent="do_exit">Exit</button>
                <!-- <button class="button is-large is-info is-outlined" @click.prevent="do_print">Print</button>
                <button class="button is-large is-info is-outlined is-danger" @click.prevent="$parent.is_refunding = true;">Refund</button> -->
                <button v-if="payment_method == 'Cash'" class="is-outlined button is-large is-warning is-active">CASH</button>
                <button v-if="payment_method == 'EFTPOS'" class="is-outlined button is-large is-primary is-active">EFTPOS</button>
                <button v-if="payment_method == 'Web Order'" class="is-outlined button is-large is-info is-active">Web Order</button>
                <button v-if="payment_method == 'Voucher'" class="is-outlined button is-large is-danger is-active">VOUCHER</button>
            </template>
            <template v-else>
                <p v-if="!$parent.has_refund_items" class="hide-in-print title is-4 has-text-danger">Please scan the item being refunded, or <a style="text-decoration: underline; display: inline-block;" class="has-text-danger" @click.prevent="do_exit">click to exit</a></p>
                <template v-else>
                    <button class="button is-large is-danger" @click.prevent="do_exit">Exit</button>
                    <button class="is-outlined button is-large is-danger is-active" @click.prevent="do_refund">REFUND: {{refund_total.toDollar()}}</button>
                </template>
            </template>
        </div>
        <div v-if="$parent.view_mode && $parent.receipt && $parent.receipt.customer" class="column hide-in-print is-narrow  align-centered">
            <p class="title is-4">{{$parent.receipt.customer.first_name[0] + '***'}} {{$parent.receipt.customer.surname}}</h2>
            <p class="subtitle is-6">{{mask_phone($parent.receipt.customer.phone)}}</p>
        </div>
        <div :class="['column has-text-right', {'is-narrow': $parent.view_mode}]">
            <p class="title is-1 discount-total">
                <button @click.prevent="$parent.cancel_discount" class="is-small is-danger button" v-if="discount">{{discount.title}}</button>
                <span class="is-tiny" v-if="discount">-{{discount.by == '%' ? (discount.rate + '%') : (discount.rate.toDollar())}}</span>
                <button v-if="$parent.coupons.length" @click.prevent="check_coupons" :class="['hide-in-print button is-danger is-circle', {'bouncy': !coupons_clicked}]"><span class="icon"><i v-if="!$parent.coupons_shown" class="fas fa-gift"></i><i class="fas fa-times" v-else></i></span></button>
                {{total_amount.toDollar()}}
            </p>
            <p class="subtitle is-7">
                incl. GST {{gst}}
                <template v-if="cash_taken"><br />Received {{cash_taken.toDollar()}} cash; Given <span class="outstanding">{{given_change}}</span> change</template>
            </p>
        </div>
    </div>
</div>
</template>

<script>
export default {
    name        :   'Summary',
    props       :   ['total', 'nondis_total', 'refund_total', 'discount', 'extra_classes', 'receipt'],
    data() {
        return {
            coupons_clicked     :   false,
            cash_loading        :   false,
            eftpos_loading      :   false,
            voucher_loading     :   false,
            refund_loading      :   false,
            web_loading         :   false,
            cash_disabled       :   false,
            eftpos_disabled     :   false,
            voucher_disabled    :   false,
            web_disabled        :   false,
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
                nondis  =   this.nondis_total ? this.nondis_total : 0,
                change  =   amount - (Math.round((this.total + nondis) * 10) * 0.1);
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

                total   =   total < 0 ? 0 : total;

                total   +=  (this.nondis_total ? this.nondis_total : 0);

                return total <= 0;
            }

            return false;
        },
        total_amount() {
            let da  =   this.total ? this.total : 0,
                nda =   this.nondis_total ? this.nondis_total : 0;

            return da + nda;
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
            me.web_loading      =   false;
            me.refund_loading   =   false;
            me.cash_disabled    =   false;
            me.web_disabled     =   false;
            me.eftpos_disabled  =   false;
            me.voucher_disabled =   false;
            me.payment_method   =   null;
            me.$router.replace('/');
            me.$parent.reset();
        },
        give_change() {
            this.payment_method =   'Cash';
            this.$bus.$emit('giveChange', this.total_amount, this.place_order);
        },
        wait_eftpos() {
            this.payment_method =   'EFTPOS';
            this.$bus.$emit('waitEFTPOS', this.total_amount, this.place_order);
        },
        wait_web() {
            if (confirm('You are placing this order as a WEB ORDER. Are you sure?')) {
                this.payment_method =   'Web Order';
                this.place_order('Web Order');
            }
        },
        voucher_payment()
        {
            this.place_order('Voucher');
        },
        do_refund()
        {
            let list    =   this.$parent.goods.filter( o => o.refund && !o.ordered);

            if (!list.length) {
                this.$bus.$emit('showMessage', 'There is nothing to be refunded', 'danger');
                return false;
            }

            if (this.refund_loading || this.cash_loading || this.eftpos_loading || this.voucher_loading || this.web_loading) return false;

            if (confirm('You are going to refund the customer: ' + this.refund_total.toDollar() + '. Do you want to proceed?')) {
                let me      =   this,
                    params  =   new FormData();

                this.payment_method =   this.receipt.method;
                this.refund_loading =   true;
                params.append('list', JSON.stringify(list));
                axios.post(
                    base_url + endpoints.refund + this.receipt.barcode,
                    params
                ).then( resp => {
                    me.$parent.do_receipt(resp.data);
                    me.$nextTick().then(() => {
                        me.$nextTick().then(() => {
                            window.print();
                            me.reset();
                        });
                    });
                }).catch( error => {
                    me.cash_loading     =   false;
                    me.eftpos_loading   =   false;
                    me.voucher_loading  =   false;
                    me.refund_loading   =   false;
                    me.web_loading      =   false;
                    me.cash_disabled    =   false;
                    me.web_disabled     =   false;
                    me.eftpos_disabled  =   false;
                    me.voucher_disabled =   false;
                    me.payment_method   =   null;
                    me.cash_taken       =   null;
                    if (error.response && error.response.data && error.response.data.message) {
                        me.$bus.$emit('showMessage', error.response.data.message);
                    }
                });
            }
        },
        place_order(by, amount_taken) {
            if (!this.$parent.goods.length) {
                this.$bus.$emit('showMessage', 'Please make sure you have scanned the items properly.', 'danger');
                return false;
            }

            if (this.refund_loading || this.cash_loading || this.eftpos_loading || this.voucher_loading || this.web_loading) return false;

            if (by == 'EFTPOS') {
                this.eftpos_loading     =   true;
            } else if (by == 'Cash'){
                this.cash_loading       =   true;
            } else if (by == 'Web Order') {
                this.web_loading        =   true;
            } else {
                this.voucher_loading    =   true;
            }

            this.web_disabled       =   true;
            this.cash_disabled      =   true;
            this.eftpos_disabled    =   true;
            this.voucher_disabled   =   true;

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
                me.web_loading      =   false;
                me.cash_loading     =   false;
                me.eftpos_loading   =   false;
                me.voucher_loading  =   false;
                me.refund_loading   =   false;
                me.web_disabled     =   false;
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
            me.voucher_loading  =   false;
            me.web_loading      =   false;
            me.refund_loading   =   false;
            me.cash_disabled    =   false;
            me.eftpos_disabled  =   false;
            me.voucher_disabled =   false;
            me.web_disabled     =   false;
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
                me.refund_loading   =   false;
                me.voucher_loading  =   false;
                me.cash_disabled    =   false;
                me.web_loading      =   false;
                me.eftpos_disabled  =   false;
                me.voucher_disabled =   false;
                me.web_disabled     =   false;
                me.payment_method   =   null;
                me.cash_taken       =   null;
                me.coupons_clicked  =   false;
                if (me.$route.name != 'Homepage') {
                    me.$router.replace('/');
                }
                me.$parent.reset();
            });
        },
        mask_phone(phone)
        {
            let rephone =   '';
            for (let i = 0; i < phone.length; i++) {
                if (i > 2 && i < phone.length - 4) {
                    rephone += '*';
                } else {
                    rephone += phone[i];
                }
            }

            return rephone;
        }
    }
}
</script>
