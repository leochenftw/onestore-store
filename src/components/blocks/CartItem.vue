<template>
<tr :class="['cart-item', {'has-text-danger' : source.refund}]">
    <td class="col-title">
        {{source.title}}
        <em v-if="source.refund" style="font-size: 14px; display: block;"><small>refund item</small></em>
        <span class="has-text-danger" v-if="!source.discountable && show_discountable" style="font-size: 14px; display: block;"><small>- not discountable -</small></span>
    </td>
    <td style="width: 15%;" class="col-price has-text-centered">{{source.unit_price.toDollar()}}</td>
    <td style="width: 12%;" class="col-qty has-text-centered">
        <input
            @dblclick.prevent="dblclick"
            @blur="blur"
            @keydown="keydown"
            type="number"
            v-model="source.quantity"
            :class="['input has-text-centered', {'has-text-danger' : source.refund}]"
            :readonly="!can_edit"
        />
    </td>
    <td style="width: 20%;" class="col-subtotal has-text-right">{{subtotal}}</td>
    <td v-if="!is_viewing" class="hide-in-print" style="white-space: nowrap; width: 62px;">
        <div class="option-more" v-if="show_options">
            <button class="button is-warning" @click.prevent="refund"><template v-if="source.refund">cancel </template>refund</button>
            <button class="button is-danger" @click.prevent="remove">remove</button>
        </div>
        <button class="button toggler is-outlined" @click.prevent="show_options = !show_options">
            <i class="fas fa-ellipsis-v" v-if="!show_options"></i>
            <i class="fas fa-times" v-else></i>
        </button>
    </td>
    <!-- <button class="button is-danger" @click.prevent="remove">remove</button> -->
</tr>
</template>

<script>
export default
{
    name        :   'CartItem',
    props       :   ['source', 'show_discountable', 'is_viewing'],
    data() {
        return {
            can_edit        :   false,
            show_options    :   false
        }
    },
    computed    :   {
        subtotal() {
            if (this.source) {
                return (this.source.unit_price * this.source.quantity * (this.source.refund ? -1 : 1)).toDollar();
            }
            return 0;
        }
    },
    mounted() {
        let me  =   this;
        this.$nextTick().then(() => {
            $('.cart-items').scrollTo($(this.$el), 300, {axis: 'y'});
        });
    },
    methods     :   {
        refund() {
            this.show_options   =   false;
            this.source.refund  =   !this.source.refund;
        },
        remove() {
            this.show_options   =   false;
            this.$parent.remove_item(this.source.id);
        },
        keydown(e) {
            if (e.keyCode == 27 || e.keyCode == 13) {
                this.blur();
            }
        },
        blur() {
            can_query       =   true;
            this.can_edit   =   false;
            $('#lookup').focus();
            if (this.source.quantity <= 0) {
                this.$parent.remove_item(this.source.id);
            }
        },
        dblclick() {
            can_query       =   false;
            this.can_edit   =   true;
            $(this.$el).find('.input').focus().select();
        }
    }
}
</script>
