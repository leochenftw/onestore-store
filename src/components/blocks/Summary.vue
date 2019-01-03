<template>
<div :class="['container summary', extra_classes]">
    <div class="columns is-mobile">
        <div class="column">
            <button :disabled="cash_disabled" @click.prevent="give_change" class="is-outlined button is-large is-warning">CASH</button>
            <button :disabled="eftpos_disabled" @click.prevent="wati_eftpos" class="is-outlined button is-large is-primary">EFTPOS</button>
        </div>
        <div class="column has-text-right">
            <p class="title is-1">
                <button class=" is-danger button" v-if="discount">{{discount.title}}</button>
                {{total_amount}}
            </p>
            <p class="subtitle is-7">incl. GST {{gst}}</p>
        </div>
    </div>
</div>
</template>

<script>
export default {
    name        :   'Summary',
    props       :   ['total', 'discount', 'extra_classes'],
    data() {
        return {
            cash_loading    :   false,
            eftpos_loading  :   false,
            cash_disabled   :   false,
            eftpos_disabled :   false
        }
    },
    computed    :   {
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
        give_change() {
            this.$bus.$emit('giveChange', this.total, this.place_order);
        },
        wati_eftpos() {
            this.$bus.$emit('waitEFTPOS', this.total, this.place_order);
        },
        place_order(by) {
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

            if (this.discount) {
                params.append('discount', this.discount.id);
            }

            axios.post(
                base_url + endpoints.order,
                params
            ).then((resp) => {
                me.$bus.$emit('showMessage', 'Thank you!', 'success', 'Completed');
                me.cash_loading     =   false;
                me.eftpos_loading   =   false;
                me.cash_disabled    =   false;
                me.eftpos_disabled  =   false;
                me.$parent.goods    =   [];
                me.$parent.discount =   null;
            }).catch((error) => {
                me.cash_loading     =   false;
                me.eftpos_loading   =   false;
                me.cash_disabled    =   false;
                me.eftpos_disabled  =   false;
                if (error.response && error.response.data && error.response.data.message) {
                    me.$bus.$emit('showMessage', error.response.data.message);
                }
            });
        },
        success_handler() {

        }
    }
}
</script>
