<template>
<div class="container summary">
    <div class="columns">
        <div class="column">
            <button class="is-outlined button is-large is-warning">CASH</button>
            <button class="is-outlined button is-large is-primary">EFTPOS</button>
        </div>
        <div class="column has-text-right">
            <p class="title is-1">{{total_amount}}</p>
            <p class="subtitle is-7">incl. GST {{gst}}</p>
        </div>
    </div>
</div>
</template>

<script>
export default {
    name        :   'Summary',
    props       :   ['total'],
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
    }
}
</script>
