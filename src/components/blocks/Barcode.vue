<template>
<svg
    v-if="barcode"
    class="barcode print-only"
    preserveAspectRatio="xMinYMin"
    :jsbarcode-value="barcode"
    jsbarcode-textmargin="0"
    jsbarcode-fontoptions="bold"
    jsbarcode-height="40"
>
</svg>
</template>

<script>
import JsBarcode from 'jsbarcode';
export default
{
    name        :   'Barcode',
    props       :   ['barcode'],
    watch       :   {
        barcode(nv, ov) {
            this.do_init();
        }
    },
    mounted() {
        this.do_init();
    },
    methods     :   {
        do_init() {
            if (this.barcode) {
                this.$nextTick().then(() => {
                    JsBarcode(".barcode").init();
                });
            }
        }
    }
}
</script>
