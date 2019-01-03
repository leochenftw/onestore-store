<template>
<div :class="['section store-face', {'has-goods' : goods.length > 0}]">
    <h1 class="title is-marginless">
        <img :class="['icon', {'is-loading': is_loading}]" src="@/assets/icon-coin.svg" />
        <img :src="logo" class="logo" alt="Store face" />
    </h1>
    <div id="stash" v-if="stash.length > 0">
        <button @click.prevent="pop_stash(item.id)" :data-id="item.id" v-for="(item, i) in stash" class="button is-danger">
            <span class="columns is-mobile">
                <span class="column">
                    <i class="fas fa-shopping-basket"></i>
                </span>
                <span class="column">
                    <span>{{translate_time(item.id)}}</span>
                    <span>ON HOLD</span>
                </span>
            </span>
        </button>
    </div>
    <button @click.prevent="put_onhold" id="btn-hold" :class="['button is-outlined is-danger', {'is-active': goods.length > 0}]"><span class="icon"><i class="far fa-hand-paper"></i></span></button>
    <StoreForm />
    <div class="container cart-items">
        <table class="table is-fullwidth">
            <thead>
                <tr>
                    <th>Product</th>
                    <th style="width: 15%;" class="has-text-centered">Unit Price</th>
                    <th style="width: 12%;" class="has-text-centered">Qty</th>
                    <th style="width: 20%;" class="has-text-right">Sub total</th>
                    <th style="width: 62px;">&nbsp;</th>
                </tr>
            </thead>
            <tbody>
                <CartItem :source="item" :show_discountable="discount" :key="i" v-for="(item, i) in goods" />
            </tbody>
        </table>
    </div>
    <Summary :total="sum" :discount="discount" :extra_classes="goods.length > 0 ? 'stand-up' : null" />
    <ChangeGiver />
    <EftposPauser />
</div>
</template>

<script>
import StoreForm from '../blocks/StoreForm';
import CartItem from '../blocks/CartItem';
import Summary from '../blocks/Summary';
import moment from 'moment';
import ChangeGiver from '../blocks/ChangeGiver';
import EftposPauser from '../blocks/EftposPauser';
export default {
    name: 'Homepage',
    components : {StoreForm, CartItem, Summary, ChangeGiver, EftposPauser},
    data() {
        return {
            logo        :   logo,
            goods       :   [],
            discount    :   null,
            is_loading  :   false,
            stash       :   []
        };
    },
    watch       :   {
        goods(nv, ov) {
            if (nv.length == 0) {
                this.discount   =   null;
            }
        }
    },
    methods     :   {
        translate_time(val) {
            return moment(val).format('HH:mm');
        },
        toggle_loading() {
            this.is_loading =   !this.is_loading;
        },
        toggle_discount(discount) {
            if (!this.goods) return;
            if (this.goods.length == 0) return;
            if (!this.discount) {
                this.discount   =   discount;
            } else if (this.discount.id == discount.id) {
                this.discount   =   null;
            } else {
                this.discount   =   discount;
            }
        },
        add_item(product) {
            let item    =   _.find(this.goods, o => o.id == product.id);
            if (item) {
                item.quantity++;
            } else {
                this.goods.push({
                    id              :   product.id,
                    title           :   product.title,
                    unit_price      :   product.price,
                    quantity        :   1,
                    refund          :   false,
                    discountable    :   product.discountable
                });
            }
        },
        remove_item(id) {
            let i   =   _.findIndex(this.goods, o => o.id == id);
            if (i >= 0) {
                this.goods.splice(i, 1);
            }
        },
        put_onhold() {
            if (this.goods.length > 0) {
                this.stash.push({
                    id      :   Date.now(),
                    list    :   _.clone(this.goods)
                });
                window.localStorage.stash   =   JSON.stringify(this.stash);
                this.goods  =   [];
            }
        },
        pop_stash(id) {
            let i       =   _.findIndex(this.stash, o => o.id == id);
            if (i >= 0) {
                let item    =   this.stash[i];
                for (let n = 0; n < item.list.length; n++) {
                    this.goods.push(item.list[n]);
                }
                this.stash.splice(i, 1);
            }
            window.localStorage.stash   =   JSON.stringify(this.stash);
        }
    },
    created() {
        if (window.localStorage.stash) {
            this.stash  =   JSON.parse(window.localStorage.stash);
        }
    },
    computed    :   {
        sum() {
            let amount          =   0,
                factor          =   1,
                nondiscountable =   0;

            if (this.discount && this.discount.type == 'byPercentage') {
                factor  -=  (this.discount.value * 0.01);
            }

            this.goods.forEach((o) => {
                if (o.discountable) {
                    amount          +=  (o.quantity * o.unit_price * factor);
                } else {
                    nondiscountable +=  (o.quantity * o.unit_price);
                }
            });

            if (this.discount && this.discount.type == 'byAmount') {
                amount -= this.discount.value;
            }

            amount  =   amount < 0 ? 0 : amount;

            return amount + nondiscountable;
        }
    }
}
</script>
