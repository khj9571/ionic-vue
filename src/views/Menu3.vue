<template>
  <div class="ion-page" main>
    <ion-header>
      <ion-toolbar>
        <span @click="openStart" slot="start">
          <MdMenuIcon w="40px" h="40px" />
        </span>
        <ion-title
          >Menu
          <span>
            <el-select
              v-model="value"
              placeholder="Select"
              @change="onRegionChange"
            >
              <el-option-group
                v-for="group in options"
                :key="group.label"
                :label="group.label"
              >
                <el-option
                  v-for="item in group.options"
                  :key="item.value"
                  :label="item.label"
                  :value="item.value"
                >
                </el-option>
              </el-option-group>
            </el-select>
          </span>
        </ion-title>
      </ion-toolbar>
    </ion-header>
    <ion-content id="content">
      <div style="padding: 10px; border: solid 1px">
        <el-form ref="form" :model="form" label-width="120px">
          <el-row>
            <el-col :span="8">
              <el-form-item label="지역구 선택">
                <el-select
                  v-model="form.selectedLocale"
                  multiple
                  collapse-tags
                  style="margin-left: 20px"
                  placeholder="Select"
                  @change="onChangeRegion"
                >
                  <el-option
                    v-for="item in localList"
                    :key="item.value"
                    :label="item.label"
                    :value="item.value"
                    :disabled="item.disabled"
                  >
                  </el-option>
                </el-select>
              </el-form-item>
            </el-col>
            <el-col :span="8">
              <el-form-item label="매물형태">
                <el-input v-model="form.name"></el-input>
              </el-form-item>
            </el-col>
            <el-col :span="8">
              <el-form-item label="가격">
                <el-slider v-model="form.price" range show-stops :max="10">
                </el-slider>
              </el-form-item>
            </el-col>
          </el-row>
        </el-form>
      </div>

      <div class="map_wrap" v-if="mapVisible">
        <vue-daum-map
          :appKey="appKey"
          :center.sync="center"
          :level.sync="level"
          :mapTypeId="mapTypeId"
          :libraries="libraries"
          @load="onLoad"
          style="
            width: 100%;
            height: 100%;
            position: relative;
            overflow: hidden;
            border: solid 1px;
          "
        />

        <div class="hAddr">
          <span class="title">지도중심기준 행정동 주소정보</span>
          <span id="centerAddr"></span>
        </div>

        <!-- <div id="menu_wrap" class="bg_white">
          <div class="option">
            <div>
              <form onsubmit="searchPlaces(); return false;">
                키워드 :
                <input type="text" value="이태원 맛집" id="keyword" size="15" />
                <button type="submit">검색하기</button>
              </form>
            </div>
          </div>
          <hr />
          <ul id="placesList"></ul>
          <div id="pagination"></div>
        </div> -->
      </div>

      <el-dialog
        title="Warning"
        :visible.sync="centerDialogVisible"
        width="30%"
        center
        append-to-body
      >
        <span
          >It should be noted that the content will not be aligned in center by
          default</span
        >
        <span slot="footer" class="dialog-footer">
          <el-button @click="centerDialogVisible = false">Cancel</el-button>
          <el-button type="primary" @click="centerDialogVisible = false"
            >Confirm</el-button
          >
        </span>
      </el-dialog>
    </ion-content>
  </div>
</template>

<script>
// import { map } from "ionicons/icons";
// import { addIcons } from "ionicons";
// addIcons({
//   "ios-map": map.ios,
//   "md-map": map.md
// });

import seoulMapInfo from "../assets/maps/seoulMap.json";
import seoulLocaleInfo from "../assets/locales/seoulLocale.json";

import MdMenuIcon from "vue-ionicons/dist/md-menu.vue";

import VueDaumMap from "vue-daum-map";

export default {
  components: {
    VueDaumMap,
    MdMenuIcon,
  },
  data: () => {
    return {
      appKey: "fc5d8d39927c118100f488a8296beb60", // 테스트용 appkey    center: { lat: 37.5411, lng: 127.068 },
      center: { lat: 37.5309123241908, lng: 127.0007353858026 }, // 지도의 중심 좌표 33.450701, 126.570667
      level: 10, // 지도의 레벨(확대, 축소 정도),127.0007353858026 /  37.5309123241908
      mapTypeId: VueDaumMap.MapTypeId.NORMAL, // 맵 타입
      libraries: ["services", "drawing"], // 추가로 불러올 라이브러리
      map: null, // 지도 객체. 지도가 로드되면 할당됨.
      markers: [],
      infowindow: null, // 인포 윈도우 참조 객체
      geocoder: null,
      menuInfowindow: null,
      displayedMenuInfowindow: false,

      mapVisible: false,
      form: {
        name: "",
        selectedLocale: [],
        price: 1,
      },
      customOverlay: null,
      polygonInfoWindow: null,
      areas: [],
      options: seoulLocaleInfo.list,
      localList: [],
      value: "",
      polygonMap: {},
      currentPolyObj: null,
      centerDialogVisible: false,
    };
  },
  methods: {
    onLoad(map) {
      this.map = map;

      this.geocoder = new kakao.maps.services.Geocoder();

      this.customOverlay = new kakao.maps.CustomOverlay({});
      this.polygonInfoWindow = new kakao.maps.InfoWindow({ removable: true });

      this.addInfo(this.map);
      //this.createInforMenu();
      this.loadMapData();

      this.areas.forEach((d) => {
        this.displayArea(d);
      });

      // 일반 지도와 스카이뷰로 지도 타입을 전환할 수 있는 지도타입 컨트롤을 생성합니다
      var mapTypeControl = new kakao.maps.MapTypeControl();
      map.addControl(mapTypeControl, kakao.maps.ControlPosition.TOPRIGHT);

      // 지도 확대 축소를 제어할 수 있는  줌 컨트롤을 생성합니다
      var zoomControl = new kakao.maps.ZoomControl();
      map.addControl(zoomControl, kakao.maps.ControlPosition.RIGHT);

      // 중심 좌표나 확대 수준이 변경됐을 때 지도 중심 좌표에 대한 주소 정보를 표시하도록 이벤트를 등록합니다
      kakao.maps.event.addListener(map, "idle", () => {
        this.searchAddrFromCoords(map.getCenter(), this.displayCenterInfo);
      });

      kakao.maps.event.addListener(map, "center_changed", function () {
        // 지도의  레벨을 얻어옵니다
        // var level = map.getLevel();
        // // 지도의 중심좌표를 얻어옵니다
        // var latlng = map.getCenter();
        // var message = ""; //"<p>지도 레벨은 " + level + " 이고</p>";
        // message +=
        //   "<p>중심 좌표는 위도 " +
        //   latlng.getLat() +
        //   ", 경도 " +
        //   latlng.getLng() +
        //   "레벨 =>" +
        //   level +
        //   "</p>";
        // var resultDiv = document.getElementById("result");
        // resultDiv.innerHTML = message;
      });
    },
    addInfo(map) {
      var marker = new kakao.maps.Marker(), // 클릭한 위치를 표시할 마커입니다
        // infowindow = new kakao.maps.InfoWindow({ zindex: 1 }); // 클릭한 위치에 대한 주소를 표시할 인포윈도우입니다

        // 커스텀 오버레이를 생성합니다
        infowindow = new kakao.maps.CustomOverlay({
          clickable: true,
          position: null,
          content: null,
          xAnchor: 0.5, // 커스텀 오버레이의 x축 위치입니다. 1에 가까울수록 왼쪽에 위치합니다. 기본값은 0.5 입니다
          yAnchor: 1.1, // 커스텀 오버레이의 y축 위치입니다. 1에 가까울수록 위쪽에 위치합니다. 기본값은 0.5 입니다
        });
      this.infowindow = infowindow;

      kakao.maps.event.addListener(map, "click", (mouseEvent) => {
        // if (this.displayedMenuInfowindow) {
        //   this.menuInfowindow.close();
        //   this.displayedMenuInfowindow = false;
        // }

        this.searchDetailAddrFromCoords(mouseEvent.latLng, (result, status) => {
          //  console.log(mouseEvent);

          if (status === kakao.maps.services.Status.OK) {
            console.log(result);

            var roadNmAddr =
              result[0].road_address != null
                ? result[0].road_address.address_name
                : "";
            var addrNm = result[0].address.address_name;

            var content = `<div class="wrap"> 
            <div class="info">
                    <div class="title"> 
                        주소
                        <div id="btnClose" class="close" @click="closeOverlay()" title="닫기"></div> 
                    </div> 
                    <div class="body"> 
                        <div class="desc"> 
                          <table>
                          <tbody>
                            <tr style="height:25px">
                              <th>도로명:</th>
                              <td><div class="ellipsis">${roadNmAddr}</div></td>
                            </tr>
                             <tr style="height:25px">
                              <th>지번:</th>
                              <td><div class="ellipsis">${addrNm}</div></td>
                            </tr>
                            <tr style="height:30px">
                              <td colspan="2" style="text-align:center">
                                <input id="btnReg" type="button" value="등록"/>
                                <input id="btnCencle" type="button" value="취소"/>
                              </td>
                            </tr>
                           </tbody>  
                          </table>
                        </div>
                    </div>
                </div>    
            </div>`;

            this.displayedMenuInfowindow = true;

            infowindow.setPosition(mouseEvent.latLng);
            infowindow.setContent(content);

            infowindow.setMap(map);

            // var detailAddr = !!result[0].road_address
            //   ? "<div>도로명주소 : " +
            //     result[0].road_address.address_name +
            //     "</div>"
            //   : "";
            // detailAddr +=
            //   "<div>지번 주소 : " + result[0].address.address_name + "</div>";

            // var content =
            //   '<div class="bAddr">' +
            //   '<span class="title">법정동 주소정보</span>' +
            //   detailAddr +
            //   "</div>";

            // // 마커를 클릭한 위치에 표시합니다 마커와 포지션 일단 주석
            marker.setPosition(mouseEvent.latLng);
            marker.setMap(map);

            this.$nextTick(() => {
              var closeBtn = document.getElementById("btnClose");
              var regBtn = document.getElementById("btnReg");
              var cencleBtn = document.getElementById("btnCencle");

              const onClose = (e) => {
                // e.stopImmediatePropagation();
                // e.stopPropagation();
                this.infowindow.setMap(null);
                this.displayedMenuInfowindow = false;
                marker.setMap(null);
              };

              closeBtn.onclick = onClose;
              cencleBtn.onclick = onClose;

              regBtn.onclick = (e) => {
                this.centerDialogVisible = true;
              };
            });
            // // 인포윈도우에 클릭한 위치에 대한 법정동 상세 주소정보를 표시합니다
            // infowindow.setContent(content);
            // infowindow.open(map, marker);
          }
        });
      });

      // 마커에 클릭이벤트를 등록합니다
      kakao.maps.event.addListener(marker, "click", () => {
        // 마커 위에 인포윈도우를 표시합니다

        if (this.displayedMenuInfowindow) {
          this.infowindow.setMap(null);
          this.displayedMenuInfowindow = false;
        } else {
          infowindow.setMap(map);
          this.displayedMenuInfowindow = true;
        }

        // if (!this.displayedMenuInfowindow) {
        //   this.menuInfowindow.open(map, marker);
        //   this.displayedMenuInfowindow = true;

        //   this.infowonodwReset();
        // } else {
        //   this.menuInfowindow.close();
        //   this.displayedMenuInfowindow = false;
        // }
      });
    },
    searchAddrFromCoords(coords, callback) {
      // 좌표로 행정동 주소 정보를 요청합니다
      this.geocoder.coord2RegionCode(
        coords.getLng(),
        coords.getLat(),
        callback
      );
    },

    searchDetailAddrFromCoords(coords, callback) {
      // 좌표로 법정동 상세 주소 정보를 요청합니다
      this.geocoder.coord2Address(coords.getLng(), coords.getLat(), callback);
    },
    // 지도 좌측상단에 지도 중심좌표에 대한 주소정보를 표출하는 함수입니다
    displayCenterInfo(result, status) {
      if (status === kakao.maps.services.Status.OK) {
        var infoDiv = document.getElementById("centerAddr");

        for (var i = 0; i < result.length; i++) {
          // 행정동의 region_type 값은 'H' 이므로
          if (result[i].region_type === "H") {
            infoDiv.innerHTML = result[i].address_name;
            break;
          }
        }
      }
    },
    // createInforMenu() {
    //   this.menuInfowindow = new kakao.maps.InfoWindow({
    //     title: "메뉴",
    //     zindex: 1,
    //   });
    //   // var content = `
    //   //   <span class="info-title">
    //   //       <ul style='list-style:none;'>
    //   //         <li>
    //   //           <input type='button' value='추가'/>
    //   //         </li>
    //   //         <li>
    //   //           <input type='button' value='삭제'/>
    //   //         </li>
    //   //         </ul>
    //   //    </span>
    //   //   `;

    //   var content = `
    //     <span class="info-title">
    //          <input type='button' value='추가'><br/>
    //          <input type='button' value='삭제'>
    //      </span>
    //   `;
    //   this.menuInfowindow.setContent(content);
    // },
    infowonodwReset() {
      var infoTitle = document.querySelectorAll(".info-title");
      infoTitle.forEach(function (e) {
        var w = e.offsetWidth + 10;
        var ml = w / 2;
        e.parentElement.style.top = "112px"; // 82
        e.parentElement.style.left = "50%";
        e.parentElement.style.marginLeft = -ml + "px";
        e.parentElement.style.width = w + "px";
        e.parentElement.previousSibling.style.display = "none";
        e.parentElement.parentElement.style.border = "0px";
        e.parentElement.parentElement.style.background = "unset";
      });
    },
    displayArea(area) {
      // 다각형을 생성합니다
      var polygon = new kakao.maps.Polygon({
        map: this.map, // 다각형을 표시할 지도 객체
        path: area.path,
        strokeWeight: 2,
        strokeColor: "#004c80",
        strokeOpacity: 0.8,
        fillColor: "#fff",
        fillOpacity: 0.1,
      });

      this.polygonMap[area.code] = polygon;

      // 다각형에 mouseover 이벤트를 등록하고 이벤트가 발생하면 폴리곤의 채움색을 변경합니다
      // 지역명을 표시하는 커스텀오버레이를 지도위에 표시합니다
      kakao.maps.event.addListener(polygon, "mouseover", (mouseEvent) => {
        polygon.setOptions({
          strokeColor: "#ff0000",
          fillColor: "#fff",
          fillOpacity: 0.1,
          strokeWeight: 5,
        });

        var content = "<div class='area'>" + area.name + "</div>";

        this.customOverlay.setContent(content);

        this.customOverlay.setPosition(mouseEvent.latLng);
        this.customOverlay.setMap(this.map);
      });

      // 다각형에 mousemove 이벤트를 등록하고 이벤트가 발생하면 커스텀 오버레이의 위치를 변경합니다
      kakao.maps.event.addListener(polygon, "mousemove", (mouseEvent) => {
        this.customOverlay.setPosition(mouseEvent.latLng);
      });

      // 다각형에 mouseout 이벤트를 등록하고 이벤트가 발생하면 폴리곤의 채움색을 원래색으로 변경합니다
      // 커스텀 오버레이를 지도에서 제거합니다
      kakao.maps.event.addListener(polygon, "mouseout", () => {
        polygon.setOptions({
          fillColor: "#fff",
          strokeColor: "#004c80",
          fillOpacity: 0.1,
          strokeWeight: 2,
        });
        this.customOverlay.setMap(null);
      });

      // 다각형에 click 이벤트를 등록하고 이벤트가 발생하면 다각형의 이름과 면적을 인포윈도우에 표시합니다
      kakao.maps.event.addListener(polygon, "click", (mouseEvent) => {
        // var content =
        //   '<div class="info">' +
        //   '   <div class="title">' +
        //   area.name +
        //   "</div>" +
        //   '   <div class="size">총 면적 : 약 ' +
        //   Math.floor(polygon.getArea()) +
        //   " m<sup>2</sup></area>" +
        //   "</div>";
        // this.polygonInfoWindow.setContent(content);
        // this.polygonInfoWindow.setPosition(mouseEvent.latLng);
        // this.polygonInfoWindow.setMap(this.map);
      });
    },
    loadMapData() {
      const areas = seoulMapInfo.features.map((d, idx) => {
        const { name, code } = d.properties;
        const [coordinates] = d.geometry.coordinates;
        const path = [];

        coordinates.forEach((c) => {
          const [x, y] = c;
          path.push(new kakao.maps.LatLng(y, x));
        });

        const obj = {
          name: name,
          code: code,
          path: path,
        };

        return obj;
      });

      this.areas = areas;
    },

    onRegionChange(d) {
      var currentObj = null;
      var isFind = this.options.some((p) => {
        const { options } = p;

        var find = options.some((c) => {
          if (c.value == d) {
            currentObj = c;
            return true;
          }
          return false;
        });

        return find;
      });

      if (!isFind) return;

      //{ lat: 37.5309123241908, lng: 127.0007353858026 }

      const { pos, code } = currentObj;

      console.log(pos);

      if (JSON.stringify(pos) == "{}") return;

      const { lat, lng } = pos;

      var moveLatLon = new kakao.maps.LatLng(lat, lng);

      this.map.setLevel(7);

      this.$nextTick(() => {
        this.map.panTo(moveLatLon);

        if (this.currentPolyObj) {
          this.currentPolyObj.setOptions({
            fillColor: "#fff",
            strokeColor: "#004c80",
            fillOpacity: 0.1,
            strokeWeight: 2,
          });
        }

        this.polygonMap[code].setOptions({
          strokeColor: "#ff0000",
          fillColor: "#fff",
          fillOpacity: 0.1,
          strokeWeight: 5,
        });

        this.currentPolyObj = this.polygonMap[code];
      });
    },
    closeOverlay() {
      alert("A");
    },
    onChangeRegion(d) {
      if (d.includes("all") || this.localList.length - 1 == d.length) {
        this.localList.forEach((d, idx) => {
          if (idx != 0) d.disabled = true;
        });
        this.form.selectedLocale = ["all"];
      } else {
        // if (this.localList.length - 1 == d.length) {
        // } else {
        this.localList.forEach((d, idx) => {
          if (idx != 0) d.disabled = false;
        });
        //}
      }

      // if (d.length == 1 && d[0] == "all") {
      //   console.log("맞음");
      //   this.localList.forEach((d, idx) => {
      //     if (idx != 0) d.disabled = true;
      //   });
      // } else if (d.length == 0) {
      //   this.localList.forEach((d, idx) => {
      //     if (idx != 0) d.disabled = false;
      //   });
      // }
    },
  },
  created() {
    var temp = [];
    seoulLocaleInfo.list.forEach((d) => {
      d.options.map((f) => {
        f.disabled = false;
        return f;
      });

      temp = temp.concat(d.options);
    });

    temp.unshift({ label: "전체", value: "all" });

    this.localList = temp;

    console.log("확인", temp);
  },
  mounted() {
    //  this.$nextTick(()=>{
    //    this.mapSize = true
    //  });
    //this.mapVisible = true

    //element.getElementById("closeBtn")

    setTimeout(() => {
      this.mapVisible = true;
    }, 100);
  },
};
</script>

<style lang="scss" scoped>
/* .map_wrap {position:relative;width:100%;height:350px;}
    .title {font-weight:bold;display:block;}
    .hAddr {position:absolute;left:10px;top:10px;border-radius: 2px;background:#fff;background:rgba(255,255,255,0.8);z-index:1;padding:5px;}
    #centerAddr {display:block;margin-top:2px;font-weight: normal;}
    .bAddr {padding:5px;text-overflow: ellipsis;overflow: hidden;white-space: nowrap;} */
.map_wrap {
  position: relative;
  width: 100%;
  height: 100%;
}
.title {
  font-weight: bold;
  display: block;
}
.hAddr {
  position: absolute;
  left: 10px;
  top: 10px;
  border-radius: 2px;
  background: #fff;
  background: rgba(255, 255, 255, 0.8);
  z-index: 1;
  padding: 5px;
}
#centerAddr {
  display: block;
  margin-top: 2px;
  font-weight: normal;
}

/deep/ {
  .bAddr {
    padding: 5px;
    text-overflow: ellipsis;
    overflow: hidden;
    white-space: nowrap;
  }

  .area {
    position: absolute;
    background: #fff;
    border: 1px solid #888;
    border-radius: 3px;
    font-size: 12px;
    top: -5px;
    left: 15px;
    padding: 2px;
  }
}

.info-title {
  display: block;
  /* background: #50627f;
  color: #fff; */
  text-align: center;
  /* height: 24px; */
  line-height: 22px;
  /* border-radius: 4px;
  padding: 0px 10px; */
}

.info {
  font-size: 12px;
  padding: 5px;
}
.info .title {
  font-weight: bold;
}

.bg-purple {
  background: #d3dce6;
}
.bg-purple-light {
  background: #e5e9f2;
}
.grid-content {
  border-radius: 4px;
  min-height: 36px;
}

.el-row {
  margin-bottom: 10px;
  &:last-child {
    margin-bottom: 0;
  }
}

/deep/ {
  .wrap {
    position: absolute;
    left: 0;
    bottom: 40px;
    width: 288px;
    height: 132px;
    margin-left: -144px;
    text-align: left;
    overflow: hidden;
    font-size: 12px;
    font-family: "Malgun Gothic", dotum, "돋움", sans-serif;
    line-height: 1.5;
  }
  .wrap * {
    padding: 0;
    margin: 0;
  }
  .wrap .info {
    width: 286px;
    height: 120px;
    border-radius: 5px;
    border-bottom: 2px solid #ccc;
    border-right: 1px solid #ccc;
    overflow: hidden;
    background: #fff;
  }
  .wrap .info:nth-child(1) {
    border: 0;
    box-shadow: 0px 1px 2px #888;
  }
  .info .title {
    padding: 5px 0 0 10px;
    height: 30px;
    background: #eee;
    border-bottom: 1px solid #ddd;
    font-size: 18px;
    font-weight: bold;
  }
  .info .close {
    position: absolute;
    top: 10px;
    right: 10px;
    color: #888;
    width: 17px;
    height: 17px;
    background: url("https://t1.daumcdn.net/localimg/localimages/07/mapapidoc/overlay_close.png");
  }
  .info .close:hover {
    cursor: pointer;
  }
  .info .body {
    position: relative;
    overflow: hidden;
  }
  .info .desc {
    position: relative;
    margin: 0px 0 0 0px;
    height: 85px;
    padding: 1px;
  }
  .desc .ellipsis {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
  .desc .jibun {
    font-size: 11px;
    color: #888;
    margin-top: -2px;
  }
  .info .img {
    position: absolute;
    top: 6px;
    left: 5px;
    width: 73px;
    height: 71px;
    border: 1px solid #ddd;
    color: #888;
    overflow: hidden;
  }
  .info:after {
    content: "";
    position: absolute;
    margin-left: -12px;
    left: 50%;
    bottom: 0;
    width: 22px;
    height: 12px;
    background: url("https://t1.daumcdn.net/localimg/localimages/07/mapapidoc/vertex_white.png");
  }
  .info .link {
    color: #5085bb;
  }

  table {
    width: 100%;
  }
  table,
  th,
  td {
    border: 1px solid #bcbcbc;
    padding: 5px;
  }
  th {
    text-align: center;
  }

  //search

  #menu_wrap {position:absolute;top:0;left:0;bottom:0;width:250px;margin:10px 0 30px 10px;padding:5px;overflow-y:auto;background:rgba(255, 255, 255, 0.7);z-index: 1;font-size:12px;border-radius: 10px;}
.bg_white {background:#fff;}
#menu_wrap hr {display: block; height: 1px;border: 0; border-top: 2px solid #5F5F5F;margin:3px 0;}
#menu_wrap .option{text-align: center;}
#menu_wrap .option p {margin:10px 0;}  
#menu_wrap .option button {margin-left:5px;}
#placesList li {list-style: none;}
#placesList .item {position:relative;border-bottom:1px solid #888;overflow: hidden;cursor: pointer;min-height: 65px;}
#placesList .item span {display: block;margin-top:4px;}
#placesList .item h5, #placesList .item .info {text-overflow: ellipsis;overflow: hidden;white-space: nowrap;}
#placesList .item .info{padding:10px 0 10px 55px;}
#placesList .info .gray {color:#8a8a8a;}
#placesList .info .jibun {padding-left:26px;background:url(https://t1.daumcdn.net/localimg/localimages/07/mapapidoc/places_jibun.png) no-repeat;}
#placesList .info .tel {color:#009900;}
#placesList .item .markerbg {float:left;position:absolute;width:36px; height:37px;margin:10px 0 0 10px;background:url(https://t1.daumcdn.net/localimg/localimages/07/mapapidoc/marker_number_blue.png) no-repeat;}
#placesList .item .marker_1 {background-position: 0 -10px;}
#placesList .item .marker_2 {background-position: 0 -56px;}
#placesList .item .marker_3 {background-position: 0 -102px}
#placesList .item .marker_4 {background-position: 0 -148px;}
#placesList .item .marker_5 {background-position: 0 -194px;}
#placesList .item .marker_6 {background-position: 0 -240px;}
#placesList .item .marker_7 {background-position: 0 -286px;}
#placesList .item .marker_8 {background-position: 0 -332px;}
#placesList .item .marker_9 {background-position: 0 -378px;}
#placesList .item .marker_10 {background-position: 0 -423px;}
#placesList .item .marker_11 {background-position: 0 -470px;}
#placesList .item .marker_12 {background-position: 0 -516px;}
#placesList .item .marker_13 {background-position: 0 -562px;}
#placesList .item .marker_14 {background-position: 0 -608px;}
#placesList .item .marker_15 {background-position: 0 -654px;}
#pagination {margin:10px auto;text-align: center;}
#pagination a {display:inline-block;margin-right:10px;}
#pagination .on {font-weight: bold; cursor: default;color:#777;}
}
</style>