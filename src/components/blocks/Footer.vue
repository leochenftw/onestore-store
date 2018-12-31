<template>
<footer id="footer" class="footer">
    <div class="container">
        <div class="has-text-centered copyright">© {{year}} Tech Bringer Ltd.</div>
    </div>
</footer>
</template>

<script>
export default {
    name: 'Footer',
    props: [

    ],
    data: function() {
        return {

        };
    },
    components: {},
    watch: {

    },
    computed: {
        year() {
            let d = new Date();

            return d.getFullYear();
        }
    }
}
</script>
