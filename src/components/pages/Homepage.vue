<template>
<div :class="['section store-face', {'has-goods' : goods.length > 0}, {'is-blurred': clookup_activated}]">
    <h1 class="title is-marginless">
        <img :class="['icon', {'is-loading': is_loading}]" src="@/assets/icon-coin.svg" />
        <img @click.prevent="focus_input" :src="logo" class="logo" alt="Store face" />
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

    <div v-if="view_mode && !is_refunding" id="top-float-menu" :class="['hide-in-print', {'is-active': goods.length > 0}]">
        <button class="button is-small is-info is-outlined" @click.prevent="do_print">Print</button>
        <button class="button is-small is-info is-outlined is-danger" @click.prevent="is_refunding = true;">Refund</button>
    </div>

    <button v-if="!view_mode" @click.prevent="toggle_customer_lookup" id="btn-call-member-input" :class="['button is-danger', {'is-active': goods.length > 0}, {'is-outlined': !clookup_activated}]"><span class="icon" v-if="customer">{{customer.first_name[0]}}</span><span class="icon" v-else><i class="fas fa-user"></i></span></button>

    <CustomerSearchForm :is_active="clookup_activated" v-if="!view_mode" />
    <StoreForm ref="storeform" v-if="!view_mode || is_refunding" />
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
                <CartItem
                    :is_viewing="view_mode"
                    :source="item"
                    :show_discountable="discount"
                    :show_pointable="receipt ? (receipt.customer ? true : false) :(customer ? true : false)"
                    :key="i" v-for="(item, i) in goods"
                />
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
    <Summary ref="summary_stripe" :receipt="receipt" :total="sum_discountable" :refund_total="refund_amount" :nondis_total="sum_nondiscountable" :discount="discount" :extra_classes="goods.length > 0 ? 'stand-up' : null" />
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
            is_refunding        :   false,
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
                this.is_refunding   =   false;
                this.view_mode      =   false;
                this.receipt        =   null;
                if (this.$route.query.receipt || this.$route.query.refund) {
                    this.$router.replace({name: 'Homepage'});
                }
            } else {
                if (this.discount && this.coupon && this.whole_cart_not_discountable()) {
                    this.coupon     =   null;
                    this.discount   =   null;
                }
            }
        }
    },
    methods     :   {
        do_print() {
            window.print();
            this.$refs.summary_stripe.slient_reset();
        },
        focus_input()
        {
            $('#lookup').focus();
        },
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
        check_qty(refund_item)
        {
            let refunding_item  =   this.goods.find(o => o.id == refund_item.id && !o.refund && o.ordered),
                refunded_item   =   this.goods.find(o => o.id == refund_item.id && o.refund && o.ordered),
                n               =   (refunded_item ? refunded_item.quantity : 0).toFloat();


            if (refunding_item.quantity < refund_item.quantity.toFloat() + n) {
                refund_item.quantity    =   refunding_item.quantity - n;
                let me  =   this;
                setTimeout(function () {
                    me.$bus.$emit('showMessage', 'You cannot refund more than you\'ve bought!', 'danger');
                }, 100);
            }
        },
        add_item(product) {
            if (this.view_mode && this.is_refunding) {
                let refunding_item  =   this.goods.find(o => o.id == product.id && !o.refund && o.ordered);

                if (refunding_item) {
                    let refund_item     =   this.goods.find(o => o.id == product.id && o.refund && !o.ordered),
                        refunded_item   =   this.goods.find(o => o.id == product.id && o.refund && o.ordered);

                    if (refund_item) {
                        if (refunding_item.quantity > refund_item.quantity + (refunded_item ? refunded_item.quantity : 0)) {
                            refund_item.quantity++;
                        } else {
                            this.$bus.$emit('showMessage', 'You cannot refund more than you\'ve bought!', 'danger');
                        }
                    } else {
                        if (refund_item || refunded_item) {
                            if (refunding_item.quantity <= (refund_item ? refund_item.quantity : 0) + (refunded_item ? refunded_item.quantity : 0)) {
                                this.$bus.$emit('showMessage', 'You cannot refund more than you\'ve bought!', 'danger');
                                return false;
                            }
                        }

                        this.goods.push({
                            id              :   product.id,
                            title           :   product.title,
                            unit_price      :   product.price,
                            quantity        :   product.quantity ? product.quantity : 1,
                            refund          :   true,
                            discountable    :   product.discountable,
                            no_point        :   product.no_point,
                            ordered         :   false
                        });
                    }
                } else {
                    this.$bus.$emit('showMessage', 'You can only refund what you\'ve bought!', 'danger');
                }

                return false;
            }

            let item    =   _.find(this.goods, (o) => {
                let b   =   product.refund ? product.refund : false;
                return o.id == product.id && o.refund == b;
            });
            if (item) {
                item.quantity += (product.quantity ? product.quantity : 1);
            } else {
                this.goods.push({
                    id              :   product.id,
                    title           :   product.title,
                    unit_price      :   product.price,
                    quantity        :   product.quantity ? product.quantity : 1,
                    refund          :   product.refund ? product.refund : false,
                    discountable    :   product.discountable,
                    no_point        :   product.no_point,
                    ordered         :   product.order_item_id ? true : false
                });
            }
        },
        remove_item(id, is_ordered, is_refund) {
            let i   =   _.findIndex(this.goods, o => o.id == id && o.ordered == is_ordered && o.refund == is_refund);
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

            if (this.$route.query.refund && this.$route.query.refund == '1') {
                this.is_refunding   =   true;
            }
        },
        reset() {
            this.goods          =   [];
            this.discount       =   null;
            this.receipt        =   null;
            this.view_mode      =   false;
            this.customer       =   null;
            this.coupon         =   null;
            this.coupons        =   [];
            this.coupons_shown  =   false;
            this.is_refunding   =   false;
        },
        toggle_coupons()
        {
            if (this.coupons.length) {
                this.coupons_shown  =   !this.coupons_shown;
            }
        },
        whole_cart_not_discountable()
        {
            let result  =   true;

            for (let i = 0; i < this.goods.length; i++) {
                if (this.goods[i].discountable) {
                    result  =   false;
                    break;
                }
            }

            return result;
        },
        create_virtual_discount(coupon, e)
        {
            if (this.whole_cart_not_discountable()) {
                alert('There is no discountable items in this cart!');
                this.toggle_coupons();
                return false;
            }

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
            if (!this.view_mode) {
                this.coupon     =   null;
                this.discount   =   null;
            }
        }
    },
    created() {
        if (window.localStorage.stash) {
            this.stash  =   JSON.parse(window.localStorage.stash);
        }
    },
    computed    :   {
        has_refund_items()
        {
            return this.goods.filter( o => o.refund && !o.ordered).length;
        },
        refund_amount()
        {
            let discountable_amount     =   0,
                nondiscountable_amount  =   0,
                factor                  =   1;

            if (this.discount && this.discount.by == '%') {
                factor  -=  (this.discount.rate * 0.01);
            }

            this.goods.forEach((o) => {
                if (o.refund && !o.ordered) {
                    if (!o.discountable) {
                        nondiscountable_amount +=  (o.quantity * o.unit_price * (o.refund ? -1 : 1));
                    } else {
                        discountable_amount +=  (o.quantity * o.unit_price * factor * (o.refund ? -1 : 1));
                    }
                }
            });

            if (this.discount && this.discount.by == '-') {
                discountable_amount -=   this.discount.rate;
                discountable_amount  =   discountable_amount < 0 ? 0 : discountable_amount;
            }

            return Math.abs(discountable_amount + nondiscountable_amount);
        },
        sum_nondiscountable()
        {
            let amount          =   0;

            this.goods.forEach((o) => {
                if (!o.discountable) {
                    amount +=  (o.quantity * o.unit_price * (o.refund ? -1 : 1));
                }
            });

            return amount;
        },
        sum_discountable()
        {
            let amount          =   0,
                factor          =   1;

            if (this.discount && this.discount.by == '%') {
                factor  -=  (this.discount.rate * 0.01);
            }

            this.goods.forEach((o) => {
                if (o.discountable) {
                    amount          +=  (o.quantity * o.unit_price * factor * (o.refund ? -1 : 1));
                }
            });

            if (this.discount && this.discount.by == '-') {
                amount -=   this.discount.rate;
                amount  =   amount < 0 ? 0 : amount;
            }

            return amount;
        }
    }
}
</script>
