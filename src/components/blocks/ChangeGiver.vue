<template>
<div :class="['modal change-giver', {'is-active': show}]">
    <div class="modal-background" @mousedown="close"></div>
    <div class="modal-content">
        <article class="message is-large is-danger">
            <div class="message-header">
                <p>Give Change</p>
            </div>
            <div class="message-body has-text-centered">
                <p class="title is-1">{{outstanding}}</p>
                <p class="subtitle is-5 has-text-info" v-if="!amount">Please enter amount</p>
                <p class="subtitle is-5 has-text-danger" v-else-if="amount < topay">Not enough</p>
                <p class="subtitle is-5 has-text-success" v-else>Change to give</p>
                <div class="field">
                    <div class="control">
                        <input id="change-giver" type="number" class="input is-large has-text-centered" v-model="amount" />
                    </div>
                </div>
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
export default
{
    name        :   'ChangeGiver',
    data() {
        return {
            show        :   false,
            topay       :   0,
            amount      :   null,
            callback    :   null
        }
    },
    computed    :   {
        outstanding() {
            let amount  =   this.amount ? this.amount : 0,
                change  =   amount - (Math.round(this.topay * 10) * 0.1);
            change      =   change > 0 ? change : 0;
            return change.toDollar();
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
        this.$bus.$on('giveChange', this.to_show);
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
                $('#change-giver').blur(function(e)
                {
                    $(this).focus();
                }).focus();
            });
        },
        do_callback() {
            if (this.callback) {
                this.callback('Cash', this.amount);
            }
            this.close();
        },
        close() {
            $('#change-giver').unbind('blur');
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
