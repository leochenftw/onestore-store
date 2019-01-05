<template>
<div :class="['container summary', extra_classes]">
    <div class="columns is-mobile">
        <div class="column" v-if="!receipt">
            <button :disabled="cash_disabled" @click.prevent="give_change" :class="['is-outlined button is-large is-warning', {'is-active': payment_method == 'Cash'}]">CASH</button>
            <button :disabled="eftpos_disabled" @click.prevent="wati_eftpos" :class="['is-outlined button is-large is-primary', {'is-active': payment_method == 'EFTPOS'}]">EFTPOS</button>
        </div>
        <div class="column" v-else>
            <button class="button is-large is-danger" @click.prevent="do_exit">Exit</button>
            <button class="button is-large is-info is-outlined" @click.prevent="do_print">Print</button>
            <button v-if="payment_method == 'Cash'" class="is-outlined button is-large is-warning is-active">CASH</button>
            <button v-if="payment_method == 'EFTPOS'" class="is-outlined button is-large is-primary is-active">EFTPOS</button>
        </div>
        <div class="column has-text-right">
            <p class="title is-1">
                <button class=" is-danger button" v-if="discount">{{discount.title}}</button>
                {{total_amount}}
            </p>
            <p class="subtitle is-7">incl. GST {{gst}}<template v-if="receipt && receipt.cash"><br />Received {{receipt.cash.toDollar()}} cash; Given {{given_change}} change</template></p>
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
            cash_loading    :   false,
            eftpos_loading  :   false,
            cash_disabled   :   false,
            eftpos_disabled :   false,
            payment_method  :   null,
            cash_taken      :   null
        }
    },
    watch       :   {
        receipt(nv, ov) {
            if (nv) {

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
            let amount  =   this.receipt.cash.toFloat() ? this.receipt.cash.toFloat() : 0,
                change  =   amount - (Math.round(this.total * 10) * 0.1);
            change      =   change > 0 ? change : 0;
            return change.toDollar();
        },
        total_amount() {
            if (this.total) {
                return this.total.toDollar();
            }

            return '$0.00';
        },
        gst() {
            if (this.total) {
                return (this.total * 0.15 / 1.15).toDollar();
            }

            return '$0.00';
        }
    },
    methods     :   {
        do_exit() {
            let me  =   this;
            me.cash_loading     =   false;
            me.eftpos_loading   =   false;
            me.cash_disabled    =   false;
            me.eftpos_disabled  =   false;
            me.payment_method   =   null;
            me.$router.replace('/');
            me.$parent.reset();
        },
        do_print() {
            window.print();
            this.reset();
        },
        give_change() {
            this.payment_method =   'Cash';
            this.$bus.$emit('giveChange', this.total, this.place_order);
        },
        wati_eftpos() {
            this.payment_method =   'EFTPOS';
            this.$bus.$emit('waitEFTPOS', this.total, this.place_order);
        },
        place_order(by, amount_taken) {
            if (this.cash_loading || this.eftpos_loading) return false;

            if (by == 'EFTPOS') {
                this.eftpos_loading     =   true;
                this.cash_disabled      =   true;
            } else {
                this.cash_loading       =   true;
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

            if (this.discount) {
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
                me.payment_method   =   null;
                me.cash_taken       =   null;
                if (error.response && error.response.data && error.response.data.message) {
                    me.$bus.$emit('showMessage', error.response.data.message);
                }
            });
        },
        reset() {
            let me  =   this;
            me.$bus.$emit('showMessage', 'Thank you!', 'success', 'Completed', () => {
                me.cash_loading     =   false;
                me.eftpos_loading   =   false;
                me.cash_disabled    =   false;
                me.eftpos_disabled  =   false;
                me.payment_method   =   null;
                me.cash_taken       =   null;
                me.$router.replace('/');
                me.$parent.reset();
            });
        }
    }
}
</script>
