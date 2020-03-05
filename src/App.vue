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
                    @change="select.area = ''")
                  option(value="") --Select One--
                  option(:value="c.CityName",v-for="c in cityName",:key="c.CityName") {{c.CityName}}
            .form-group.d-flex
              label.mr-2.col-form-label.text-right(for='area') 地區
              .flex-fill
                select#area.form-control(
                    v-model="select.area", v-if="select.city.length"
                    @change="removeMarker(); updateMap()")
                  option(value='') -- Select One --
                  option(:value="a.AreaName",
                      v-for="a in cityName.find((city) => city.CityName === select.city).AreaList",
                         :key="a.AreaName") {{a.AreaName}}
            p.mb-0.small.text-muted.text-left 選擇區域查看（綠色表示兩種口罩尚有庫存）
          ul.list-group
            template(v-for="(item, key) in data",
                  v-if='item.properties.county === select.city\
                  && item.properties.town === select.area')
              a.list-group-item.text-left(:key='key',
                  v-if='item.properties.county === select.city\
                  && item.properties.town === select.area',
                  :class="{ 'highlight': item.properties.mask_adult >0\
                            && item.properties.mask_child >0 }",
                  @click='penTo(item)')
                h3 {{item.properties.name}}
                p.mb-0
                  | 成人口罩：{{item.properties.mask_adult}} | 兒童口罩：{{item.properties.mask_child}}
                p.mb-0 電話： {{item.properties.phone}}
                p.mb-0(:class="{ 'highlight': item.properties.mask_adult >0\
                          && item.properties.mask_child >0 }") 地址：
                  a(:href='`https://www.google.com.tw/maps/place/${item.properties.address}`',
                    target='_blank', title='Google Map') {{item.properties.address}}
                p.mb-0 備註： {{item.properties.note}}
      .col-sm-9
        #map

</template>

// https://github.com/Wcc723/wheremask/tree/master/src
// https://paper.dropbox.com/doc/2020-0213-2XPGbqH5JqE9vTD7a6npd

<script>
import L from 'leaflet';
import cityName from './assets/cityName.json';

let osmMap = {};

const iconsConfig = {
  iconSize: [25, 41],
  iconAnchor: [12, 41],
  popupAnchor: [1, -34],
  shadowSize: [41, 41],
};
const icons = {
  green: new L.Icon({
    iconUrl: 'https://cdn.rawgit.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-green.png',
    ...iconsConfig,
  }),
  grey: new L.Icon({
    iconUrl: 'https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-grey.png',
    ...iconsConfig,
  }),
};

export default {
  name: 'App',
  data: () => ({
    data: [],
    cityName,
    select: {
      city: '臺北市',
      area: '中正區',
    },
  }),
  // watch: {
  //   select: {
  //     handler() {
  //       this.select.area = cityName.find((city) => (
  //         city.CityName === this.select.city).AreaList[0].AreaName);
  //     },
  //     deep: true,
  //   },
  // },
  methods: {
    updateMap() {
      const pharmacies = this.data.filter((pharmacy) => (
        pharmacy.properties.county === this.select.city)
        && (pharmacy.properties.town === this.select.area));

      pharmacies.forEach((pharmacy) => {
        const { properties, geometry } = pharmacy;
        const icon = properties.mask_adult > 0
        && properties.mask_child > 0 ? icons.green : icons.grey;
        L.marker([
          geometry.coordinates[1],
          geometry.coordinates[0],
        ], { icon }).addTo(osmMap).bindPopup(`
         <H6><strong>${properties.name}</strong></H6>  
         口罩數量：<strong>
         成人 - ${properties.mask_adult ? `${properties.mask_adult} 個` : '無庫存'}/ 
         兒童 - ${properties.mask_child ? `${properties.mask_child} 個` : '無庫存'}</strong><br/>    
         地址: <a href="https://www.google.com.tw/maps/place/${properties.address}" target="_blank">${properties.address}</a><br/>    
         電話: ${properties.phone}<br/>
         備註: ${properties.note}<br/>
         <small>最後更新時間: ${properties.updated}</small><br/>
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
      osmMap.panTo([geometry.coordinates[1], geometry.coordinates[0]]);
      const icon = properties.mask_adult > 0
      && properties.mask_child > 0 ? icons.green : icons.grey;
      L.marker([
        geometry.coordinates[1],
        geometry.coordinates[0],
      ], { icon }).addTo(osmMap).bindPopup(`
        <H6><strong>${properties.name}</strong></H6>  
        口罩數量：<strong>
        成人 - ${properties.mask_adult ? `${properties.mask_adult} 個` : '無庫存'}/ 
        兒童 - ${properties.mask_child ? `${properties.mask_child} 個` : '無庫存'}</strong><br/>    
        地址: <a href="https://www.google.com.tw/maps/place/${properties.address}" target="_blank">${properties.address}</a><br/>    
        電話: ${properties.phone}<br/>
        備註: ${properties.note}<br/>
        <small>最後更新時間: ${properties.updated}</small><br/>
        `).openPopup();
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
