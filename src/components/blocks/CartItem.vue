<template>
<tr class="cart-item">
    <td><button class="button is-danger" @click.prevent="remove">remove</button></td>
    <td>{{source.title}}</td>
    <td class="has-text-centered">{{source.unit_price.toDollar()}}</td>
    <td class="has-text-centered">
        <input type="text" v-model="source.quantity" class="input has-text-centered" size="1" />
    </td>
    <td class="has-text-right">{{subtotal}}</td>
</tr>
</template>

<script>
export default
{
    name        :   'CartItem',
    props       :   ['source'],
    computed    :   {
        subtotal() {
            if (this.source) {
                return (this.source.unit_price * this.source.quantity).toDollar();
            }
            return 0;
        }
    },
    methods     :   {
        remove() {
            this.source.quantity--;
            if (this.source.quantity <= 0) {
                this.$parent.remove_item(this.source.id);
            }
        }
    }
}
</script>
