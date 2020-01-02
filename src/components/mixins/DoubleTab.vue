<template></template>

<script>
export default {
    name    :   'DoubleTab',
    data()
    {
        return {
            first_trigger   :   false
        }
    },
    methods :   {
        mount_listener()
        {
            $(window).on('keydown', this.double_tab_handler);
        },
        double_tab_handler(e)
        {
            if (e.keyCode == 13) {
                let me  =   this;
                if (!this.first_trigger) {
                    this.first_trigger  =   true;
                    setTimeout(function () {
                        me.first_trigger    =   false;
                    }, 600);

                    return false;
                }

                this.do_callback();
            }
        },
        dismount_listener()
        {
            this.first_trigger  =   false;
            $(window).off('keydown', this.double_tab_handler);
        }
    }
}
</script>
