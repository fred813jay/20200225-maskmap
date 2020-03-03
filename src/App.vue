<template lang="pug">
  #app
    .row.no-gutters
      .col-sm-3
        .toolbox
          .sticky-top.bg-white.shadow-sm.p-2
            .form-group.d-flex
              label.mr-2.col-form-label.text-right(for='cityName') 縣市
              .flex-fill
                select#cityName.form-control(
                    v-model="select.city",
                    @change="removeMarker(); updateMap()")
                  option(value="") --Select One--
                  option(:value="c.CityName",v-for="c in cityName",:key="c.CityName") {{c.CityName}}
            .form-group.d-flex
              label.mr-2.col-form-label.text-right(for='area') 地區
              .flex-fill
                select#area.form-control
                  option(value='') -- Select One --
            p.mb-0.small.text-muted.text-right 請先選擇區域查看（綠色表示還有口罩）
          ul.list-group
            template
              a.list-group-item.text-left
                h3 藥局名稱
                p.mb-0
                  | 成人口罩：1 | 兒童口罩：2
              p.mb-0
                a.list-group-item.text-left 地址：
                  a(href='https://www.google.com.tw/maps/place/...', target='_blank', title='Google Map') 地址
      .col-sm-9
        #map

</template>

// https://github.com/Wcc723/wheremask/tree/master/src
// https://paper.dropbox.com/doc/2020-0213-2XPGbqH5JqE9vTD7a6npd

<script>
import L from 'leaflet';
import cityName from './assets/cityName.json';

let osmMap = {};

// console.log(L);

export default {
  name: 'App',
  data: () => ({
    data: [],
    cityName,
    select: {
      city: '臺北市',
    },
  }),
  methods: {
    updateMap() {
      const pharmacies = this.data.filter((pharmacy) => (
        pharmacy.properties.county === this.select.city));
      pharmacies.forEach((pharmacy) => {
        const { properties, geometry } = pharmacy;
        L.marker([
          geometry.coordinates[1],
          geometry.coordinates[0],
        ]).addTo(osmMap).bindPopup(`
         <H6><strong>${properties.name}</strong></H6>  
         口罩數量：<strong>
         成人 - ${properties.mask_adult ? `${properties.mask_adult} 個` : '無庫存'}/ 
         兒童 - ${properties.mask_child ? `${properties.mask_child} 個` : '無庫存'}</strong><br/>    
         地址: <a href="https://www.google.com.tw/maps/place/${properties.address}" target="_blank">${properties.address}</a><br/>    
         電話: ${properties.phone}<br/><small>最後更新時間: ${properties.updated}</small>
          `);
      });
      this.penTo(pharmacies[0]);
    },
    // 每次更換區域前先清除地圖標示
    removeMarker() {
      osmMap.eachLayer((layer) => {
        if (layer instanceof L.Marker) {
          osmMap.removeLayer(layer);
        }
      });
    },
    // 選擇地區後將地圖中心點帶入座標
    penTo(item) {
      const { properties, geometry } = item;
      console.log(properties);
      osmMap.panTo([geometry.coordinates[1], geometry.coordinates[0]]);
    },
  },
  mounted() {
    const url = 'https://raw.githubusercontent.com/kiang/pharmacies/master/json/points.json';
    // 取得API(axios)
    this.$http.get(url).then((response) => {
      this.data = response.data.features;
      this.updateMap();
    });
    // 定義地圖起始位置
    osmMap = L.map('map', {
      center: [25.03, 121.55],
      zoom: 15,
    });
    // 導入圖資
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: 'Map data &copy; <a href="https://www.openstreetmap.org/">OpenStreetMap</a> contributors,<a href="https://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>',
      maxZoom: 18,
    }).addTo(osmMap);
  },
};
</script>


<style lang="sass">
@import 'bootstrap/scss/bootstrap'
#map
  height: 100vh
.highlight
  background: #e9ffe3
.toolbox
  height: 100vh
  overflow-y: auto
  a
    cursor: pointer
</style>
