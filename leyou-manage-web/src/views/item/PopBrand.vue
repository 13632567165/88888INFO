<template>
    <div>
        <v-layout row wrap>
            <v-flex xs6>
                <v-btn color="primary" round>品牌添加</v-btn>
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
                <td class="text-xs-center"><img :src="props.item.image"> </img></td>
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
                    {text: '品牌ID', align: "center", value: 'id'},
                    {text: '品牌名称', align: "center", sortable: false, value: 'name'},
                    {text: '品牌图片', align: "center", sortable: false, value: 'image'},
                    {text: '品牌首字母', align: "center", value: 'letter'},
                ]
            }
        },
        watch: {
            "searchKey":{
                handler() {
                    this.getBrandData();
                }
            }
            ,
            "pagination": {
                deep: true,
                handler() {
                    this.getBrandData();
                }
            }
        },
        mounted() {
            this.getBrandData();
        },
        methods: {
            getBrandData() {
                this.$http.get("/item/brand/page", {
                    params: {
                        page: this.pagination.page,
                        rows: this.pagination.rowsPerPage,
                        sortBy: this.pagination.sortBy,
                        desc: this.pagination.descending,
                        key: this.searchKey
                    }
                }).then(resp => {
                    this.desserts = resp.data.items
                    this.totalDesserts = resp.data.total
                }).catch(e => {
                    console.log("品牌查询失败");
                })

            },
            getDesserts() {
                return [
                    {id: 2032, name: "OPPO", image: "1.jpg", letter: "O"},
                    {id: 2033, name: "飞利浦", image: "2.jpg", letter: "F"},
                    {id: 2034, name: "华为", image: "3.jpg", letter: "H"},
                    {id: 2036, name: "酷派", image: "4.jpg", letter: "K"},
                    {id: 2037, name: "魅族", image: "5.jpg", letter: "M"}
                ]
            }
        }
    };
</script>
