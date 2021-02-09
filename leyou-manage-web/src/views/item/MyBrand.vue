<template>
    <div>
        <v-layout row wrap>
            <v-flex xs6>
                <v-btn round color="primary" dark>品牌添加</v-btn>
            </v-flex>

            <v-flex xs6>
                <v-text-field
                        label="搜索"
                        prepend-icon="search"
                        v-model="searchKey"
                ></v-text-field>
            </v-flex>
        </v-layout>
        <v-data-table
                :headers="headers"
                :items="desserts"
                :pagination.sync="pagination"
                :total-items="totalDesserts"
                :loading="loading"
                class="elevation-1"
        >
            <template v-slot:items="props">
                <td class="text-xs-center">{{ props.item.id }}</td>
                <td class="text-xs-center">{{ props.item.name }}</td>
                <td class="text-xs-center"><img :src="props.item.image"/></td>
                <td class="text-xs-center">{{ props.item.letter }}</td>
            </template>
        </v-data-table>
    </div>
</template>
<script>
    export default {
        data() {
            return {
                totalDesserts: 0,
                desserts: [],
                loading: true,
                pagination: {},
                searchKey: '',
                headers: [
                    {text: '品牌编号', align: 'center', value: 'id'},
                    {text: '品牌名称', sortable: false, align: 'center', value: 'name'},
                    {text: '品牌图片', sortable: false, align: 'center', value: 'image'},
                    {text: '品牌字母', align: 'center', value: 'letter'}
                ]
            };
        },
        watch: {
            "searchKey": {
                handler() {
                    this.loadBrandData();
                }
            },
            "pagination": {
                deep: true,
                handler() {
                    this.loadBrandData()
                }
            }
        },
        mounted() {
            this.loadBrandData();
        },
        methods: {
            loadBrandData() {
                this.$http.get('/item/brand/page', {
                    params: {
                        page: this.pagination.page,
                        rows: this.pagination.rowsPerPage,
                        sortBy: this.pagination.sortBy,
                        desc: this.pagination.descending,
                        key: this.searchKey
                    }
                }).then(resp => {
                    this.desserts = resp.data.items;
                    this.totalDesserts = resp.data.total;
                    this.loading = false;
                }).catch(e => {
                    console.log('品牌查询失败');
                });

            }
        }
    };
</script>
