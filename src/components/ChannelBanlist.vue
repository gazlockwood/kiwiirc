<template>
    <div class="kiwi-channelbanlist">
        <form class="u-form kiwi-channelbanlist" @submit.prevent="">
            <a @click="updateBanlist" class="u-link">Refresh banlist</a>

            <table v-if="banlist.length > 0" class="kiwi-channelbanlist-table">
                <tr>
                    <th>Banned user</th>
                    <th>Banned by</th>
                    <th></th>
                    <th></th>
                </tr>
                <tr v-for="ban in banlist">
                    <td class="kiwi-channelbanlist-table-mask">{{ban.banned}}</td>
                    <td class="kiwi-channelbanlist-table-bannedby">{{ban.banned_by}}</td>
                    <td class="kiwi-channelbanlist-table-bannedat">{{(new Date(ban.banned_at * 1000)).toDateString()}}</td>
                    <td class="kiwi-channelbanlist-table-actions"><i @click="removeBan(ban.banned)" class="fa fa-trash" aria-hidden="true"></i></td>
                </tr>
            </table>
            <div v-else-if="is_refreshing">
                Refreshing banlist...
            </div>
            <div v-else class="kiwi-channelbanlist-empty">
                Nobody is banned!
            </div>
        </form>
    </div>
</template>

<script>

export default {
    data: function data() {
        return {
            banlist: [],
            is_refreshing: false,
        };
    },
    props: ['buffer'],
    methods: {
        updateBanlist: function updateBanlist() {
            if (this.buffer.getNetwork().state !== 'connected' || this.is_refreshing) {
                return;
            }

            let channelName = this.buffer.name;
            this.is_refreshing = true;
            this.buffer.getNetwork().ircClient.banlist(channelName, banEvent => {
                this.banlist = banEvent.bans;
                this.is_refreshing = false;
            });
        },
        removeBan: function removeBan(mask) {
            let channelName = this.buffer.name;
            this.buffer.getNetwork().ircClient.unban(channelName, mask);
            this.banlist = this.banlist.filter(ban => ban.banned !== mask);
        },
    },
    created: function created() {
        this.updateBanlist();
    },
};

</script>

<style>
.kiwi-channelbanlist-table {
    width: 100%;
    border-collapse: collapse;
}
.kiwi-channelbanlist-table tr {
    border-bottom: 1px solid #dddddd;
}
</style>
