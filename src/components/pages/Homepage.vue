<template>
<div :class="['section store-face', {'has-goods' : goods.length > 0}, {'is-blurred': clookup_activated}]">
    <h1 class="title is-marginless">
        <img :class="['icon', {'is-loading': is_loading}]" src="@/assets/icon-coin.svg" />
        <img :src="logo" class="logo" alt="Store face" />
        <div id="store-info" class="has-text-right" v-if="store_info">
            <p v-if="store_info.title">{{store_info.title}}</p>
            <p v-if="store_info.gst">GST: {{store_info.gst}}</p>
            <p v-if="store_info.phone">{{store_info.phone}}</p>
        </div>
    </h1>
    <div id="stash" v-if="stash.length > 0 && !view_mode">
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
    <button v-if="!view_mode" @click.prevent="put_onhold" id="btn-hold" :class="['button is-outlined is-danger', {'is-active': goods.length > 0}]"><span class="icon"><i class="far fa-hand-paper"></i></span></button>

    <button v-if="!view_mode" @click.prevent="toggle_customer_lookup" id="btn-call-member-input" :class="['button is-danger', {'is-active': goods.length > 0}, {'is-outlined': !clookup_activated}]"><span class="icon" v-if="customer">{{customer.first_name[0]}}</span><span class="icon" v-else><i class="fas fa-user"></i></span></button>

    <CustomerSearchForm :is_active="clookup_activated" v-if="!view_mode" />
    <StoreForm ref="storeform" v-if="!view_mode" />
    <div class="container cart-items">
        <table class="table is-fullwidth">
            <thead>
                <tr>
                    <th>Product</th>
                    <th style="width: 15%;" class="has-text-centered">Unit Price</th>
                    <th style="width: 12%;" class="has-text-centered">Qty</th>
                    <th style="width: 20%;" class="has-text-right">Sub total</th>
                    <th v-if="!view_mode" class="hide-in-print" style="width: 62px;">&nbsp;</th>
                </tr>
            </thead>
            <tbody>
                <CartItem :is_viewing="view_mode" :source="item" :show_discountable="discount" :key="i" v-for="(item, i) in goods" />
            </tbody>
        </table>
    </div>

    <div class="container">
        <ul id="coupon-list" v-if="coupons.length" :class="{'show': coupons_shown}">
            <li v-for="item in coupons">
                <button @click.prevent="create_virtual_discount(item, $event)" class="button is-danger">
                    {{item.title}}
                    <i :class="['far fa-check-circle', {'is-visible': (coupon && coupon.id == item.id)}]"></i>
                </button>
            </li>
        </ul>
    </div>

    <div id="receipt-misc" v-if="receipt">
        <p><strong>NOTE: </strong>ALL PRICES ARE GST INCLUSIVE</p>
        <p><strong>Operator: </strong>{{receipt.by}}</p>
        <p><strong>Paid at: </strong><span id="paid-at">{{receipt.at}}</span></p>
        <template v-if="receipt.customer">
        <br />
        <p><strong>Customer: </strong>{{receipt.customer.first_name}} {{receipt.customer.surname}}</p>
        <p><strong>Phone: </strong>{{receipt.customer.phone}}</p>
        <p><strong>Credit: </strong>{{receipt.customer.shop_points.kmark()}} (as at {{receipt.at}})</p>
        </template>
    </div>
    <Summary :receipt="receipt" :total="sum" :discount="discount" :extra_classes="goods.length > 0 ? 'stand-up' : null" />
    <ChangeGiver />
    <EftposPauser />
    <Barcode v-if="receipt" :barcode="receipt.barcode" />
</div>
</template>

<script>
import StoreForm from '../blocks/StoreForm';
import CartItem from '../blocks/CartItem';
import Summary from '../blocks/Summary';
import moment from 'moment';
import ChangeGiver from '../blocks/ChangeGiver';
import EftposPauser from '../blocks/EftposPauser';
import Barcode from '../blocks/Barcode';
import CustomerSearchForm from '../blocks/CustomerSearchForm';

export default {
    name: 'Homepage',
    components : {StoreForm, CartItem, Summary, ChangeGiver, EftposPauser, Barcode, CustomerSearchForm},
    data() {
        return {
            view_mode           :   false,
            store_info          :   store_info,
            logo                :   logo,
            goods               :   [],
            discount            :   null,
            is_loading          :   false,
            stash               :   [],
            receipt             :   null,
            clookup_activated   :   false,
            customer            :   null,
            coupon              :   null,
            coupons             :   [],
            coupons_shown       :   false
        };
    },
    watch       :   {
        goods(nv, ov) {
            if (nv.length == 0) {
                this.discount       =   null;
                this.customer       =   null;
                this.coupons        =   [];
                this.coupon         =   null;
                this.coupons_shown  =   false;
            }
        }
    },
    methods     :   {
        toggle_customer_lookup(e)
        {
            can_query               =   !can_query;
            this.clookup_activated  =   !this.clookup_activated;
            if (can_query) {
                this.$nextTick().then(() => {
                    this.$refs.storeform.focusing();
                });
            }
        },
        do_receipt(data) {
            this.receipt    =   data;
        },
        translate_time(val) {
            return moment(val).format('HH:mm');
        },
        toggle_loading() {
            this.is_loading =   !this.is_loading;
        },
        toggle_discount(discount) {
            if (!this.goods) return;
            if (this.goods.length == 0) return;
            if (this.coupon) {
                this.coupon     =   null;
                this.discount   =   discount;
            } else {
                if (!this.discount) {
                    this.discount   =   discount;
                } else if (this.discount.id == discount.id) {
                    this.discount   =   null;
                } else {
                    this.discount   =   discount;
                }
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
                    quantity        :   product.quantity ? product.quantity : 1,
                    refund          :   product.refund ? parseInt(product.refund) : parseInt(product.refund),
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
                    list    :   _.clone(this.goods),
                    customer:   this.customer,
                    coupons :   this.coupons,
                    coupon  :   this.coupon
                });
                window.localStorage.stash   =   JSON.stringify(this.stash);
                this.goods          =   [];
                this.customer       =   null;
                this.coupons        =   [];
                this.coupon         =   null;
                this.coupons_shown  =   false;
            }
        },
        pop_stash(id) {
            let i       =   _.findIndex(this.stash, o => o.id == id);
            if (i >= 0) {
                let item        =   this.stash[i];
                this.customer   =   item.customer;
                this.coupons    =   item.coupons;
                this.coupon     =   item.coupon;
                if (this.coupon) {
                    this.create_virtual_discount(this.coupon);
                }
                for (let n = 0; n < item.list.length; n++) {
                    this.goods.push(item.list[n]);
                }
                this.stash.splice(i, 1);
            }
            window.localStorage.stash   =   JSON.stringify(this.stash);
        },
        reprint_receipt(receipt) {
            this.view_mode  =   true;
            if (this.goods.length > 0) {
                this.put_onhold();
            }

            if (receipt.goods) {
                let me  =   this;
                receipt.goods.forEach((o) => {
                    me.add_item(o);
                });
            }

            if (receipt.discount) {
                this.toggle_discount(receipt.discount);
            }

            if (receipt.order) {
                this.do_receipt(receipt.order);
                this.$bus.$emit('onMethdDominated', receipt.order.method);
            }
        },
        reset() {
            this.goods          =   [];
            this.discount       =   null;
            this.receipt        =   null;
            this.view_mode      =   false;
            this.customer       =   null;
            this.coupon         =   null;
            this.coupons_shown  =   false;
        },
        toggle_coupons()
        {
            if (this.coupons.length) {
                this.coupons_shown  =   !this.coupons_shown;
            }
        },
        create_virtual_discount(coupon, e)
        {
            this.coupon     =   coupon;
            this.discount   =   {
                id: coupon.id,
                title: coupon.title,
                by: '-',
                rate: coupon.worth
            };
            if (e) {
                this.toggle_coupons();
            }
        },
        cancel_discount(e)
        {
            this.coupon     =   null;
            this.discount   =   null;
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
                    amount          +=  (o.quantity * o.unit_price * factor * (o.refund ? -1 : 1));
                } else {
                    nondiscountable +=  (o.quantity * o.unit_price * (o.refund ? -1 : 1));
                }
            });

            if (this.discount && this.discount.type == 'byAmount') {
                amount -=   this.discount.value;
                amount  =   amount < 0 ? 0 : amount;
            }

            return amount + nondiscountable;
        }
    }
}
</script>
