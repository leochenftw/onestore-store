<template>
<div :class="['modal eftpos-pauser', {'is-active': show}]">
    <div class="modal-background" @mousedown="close"></div>
    <div class="modal-content">
        <article class="message is-large is-danger">
            <div class="message-header">
                <p>EFTPOS</p>
            </div>
            <div class="message-body has-text-centered">
                <p class="title is-1">{{topay.toDollar()}}</p>
                <p class="subtitle is-5 has-text-danger"><strong>OK</strong> when the transaction has gone through</p>
            </div>
            <div class="message-foot has-text-centered">
                <button @click.prevent="close" class="button is-outlined is-large">Cancel</button>
                <button @click.prevent="do_callback" class="button is-large is-danger">OK</button>
            </div>
        </article>
    </div>
</div>
</template>

<script>
import DoubleTab from '../mixins/DoubleTab';
export default
{
    name        :   'EftposPauser',
    mixins      :   [DoubleTab],
    data() {
        return {
            show        :   false,
            topay       :   0,
            callback    :   null
        }
    },
    watch       :   {
        show(nv, ov) {
            if (nv == true) {
                $(window).bind('keydown', this.keydownhandler);
            }
        }
    },
    created() {
        this.$bus.$on('waitEFTPOS', this.to_show);
    },
    methods     :   {
        keydownhandler(e) {
            if (e.keyCode == 27) {
                this.close();
            }
        },
        to_show(topay, callback) {
            this.show       =   true;
            this.topay      =   topay;
            this.callback   =   callback;

            can_query       =   false;
            $('#lookup').blur();
            this.$nextTick().then(() => {
                this.mount_listener();
            });
        },
        do_callback() {
            if (this.callback) {
                this.callback('EFTPOS');
            }
            this.close();
        },
        close() {
            this.dismount_listener();
            $(window).unbind('keydown', this.keydownhandler);
            this.show       =   false;
            this.topay      =   0;
            this.amount     =   null;
            this.callback   =   null;
            can_query       =   true;
            $('#lookup').focus();
        }
    }
}
</script>
