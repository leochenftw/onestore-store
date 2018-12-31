<template>
<div :class="['section store-face', {'has-goods' : goods.length > 0}]">
    <h1 class="title is-marginless">
        <img :class="['icon', {'is-loading': is_loading}]" src="@/assets/icon-coin.svg" />
        <img :src="logo" class="logo" alt="Store face" />
    </h1>
    <div class="container cart-items">
        <table class="table is-fullwidth">
            <thead>
                <tr>
                    <th style="width: 80px;"></th>
                    <th>Product</th>
                    <th class="has-text-centered">Unit Price</th>
                    <th class="has-text-centered">Qty</th>
                    <th class="has-text-right">Sub total</th>
                </tr>
            </thead>
            <tbody>
                <CartItem :source="item" :key="i" v-for="(item, i) in goods" />
            </tbody>
        </table>
    </div>
    <Summary :total="sum" />
    <StoreForm />
</div>
</template>

<script>
import StoreForm from '../blocks/StoreForm';
import CartItem from '../blocks/CartItem';
import Summary from '../blocks/Summary';
export default {
    name: 'Homepage',
    components : {StoreForm, CartItem,Summary},
    data() {
        return {
            logo        :   logo,
            goods       :   [],
            is_loading  :   false
        };
    },
    methods     :   {
        toggle_loading() {
            this.is_loading =   !this.is_loading;
        },
        deduct(discount) {

        },
        add_item(product) {
            let item    =   _.find(this.goods, o => o.id == product.id);
            if (item) {
                item.quantity++;
            } else {
                this.goods.push({
                    id          :   product.id,
                    title       :   product.title,
                    unit_price  :   product.price,
                    quantity    :   1
                });
            }
        },
        remove_item(id) {
            let i   =   _.findIndex(this.goods, o => o.id == id);
            this.goods.splice(i, 1);
        }
    },
    computed    :   {
        sum() {
            let amount  =   0;
            this.goods.forEach((o) => {
                amount += (o.quantity * o.unit_price);
            });

            return amount;
        }
    }
}
</script>
