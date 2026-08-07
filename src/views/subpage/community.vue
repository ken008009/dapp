<template>
<div class='shop-page'>
  <van-nav-bar
    :title="lang('社群建设')"
    left-arrow
    :border="false"
    fixed
    @click-left="handleBack"
  />
  <div class="page-main">
    <div class="page-title">
      <h2>{{ lang('我的社群') }}</h2>
      <p>{{ lang('您可以在这里绑定邀请关系，也可以复制邀请链接邀请您的好友加入。') }}</p>
    </div>
    <div className="info-box">
      <div className="info-title">{{ lang('上级邀请地址') }}</div>
      <div className="info-address">{{ formatAddress(userinfo.inviteUserAddress) }}</div>
    </div>
    <div className="info-box">
      <div className="info-title">{{ lang('我的邀请链接') }}</div>
      <div className="info-link">
        <span>{{ inviteUrl }}</span>
        <i className="copy-address-button" @click="copyToClipboard(inviteUrl)"></i>
      </div>
    </div>
    <div className="performance-list">
      <div className="performance-info">
        <div className="performance-info-item">
          <p>{{ userinfo.total || 0 }}</p>
          <p>{{ lang('总业绩') }}</p>
        </div>
        <div className="performance-info-item">
          <p>{{ userinfo.max || 0 }}</p>
          <p>{{ lang('大区业绩') }}</p>
        </div>
      </div>
      <div className="performance-share-title">{{ lang('直接邀请数据') }}</div>
      <div className="performance-share-list">
        <a-tree
          v-if="treeData.length > 0"
          v-model:expandedKeys="expandedKeys"
          v-model:selectedKeys="selectedKeys"
          :load-data="onLoadData"
          :tree-data="treeData"
        />
        <!-- <div className="performance-share-header">
          <span>{{ lang('地址') }}</span>
          <span>{{ lang('个人累计') }}</span>
          <span>{{ lang('团队累计') }}</span>
        </div>
        <div className="performance-share-item">
        </div> -->
      </div>
    </div>
    <div className="team-deposit-list">
      <div className="team-deposit-title">{{ lang('团队充值') }} ({{ depositCount }}{{ lang('条') }})</div>
      <div className="team-deposit-table">
        <div className="table-header" v-if="depositList.length > 0">
          <div className="table-row">
            <span>{{ lang('地址') }}</span>
            <span>{{ lang('金额') }}</span>
            <span>{{ lang('日期') }}</span>
          </div>
        </div>
        <div className="table-body">
          <div className="table-row" v-for="(item, index) in depositList" :key="index">
            <span>{{ formatAddress(item.address) }}</span>
            <span>{{ item.amount }}</span>
            <span>{{ item.createdAt }}</span>
          </div>
        </div>
        <div className="empty" v-if="depositList.length === 0">
          <img :src="emptyImage" />
          <div className="empty-text">{{ lang('暂无数据') }}</div>
        </div>
      </div>
      <div className="team-deposit-pagination" v-if="depositList.length > 0">
        <!-- <span className="page-size-text">
          {{ lang('每页') }} {{ depositPageSize }}{{ lang('条') }} · {{ lang('本页') }} {{ depositList.length }}{{ lang('条') }}
        </span> -->
        <Pagination
          v-model="depositPage"
          :page-count="depositAllPageCount"
          mode="simple"
          @change="getTeamDepositList"
        />
      </div>
    </div>
  </div>
</div>
</template>
<script setup>
import userPerson from "@/pinia/person";
import { useRouter } from 'vue-router'
import { ref, computed } from 'vue'
import { showToast, Pagination } from 'vant'
import request from "@/tools/request";
import lang from '@/i18n/index'
import copy from 'copy-to-clipboard';
import emptyImage from '../../assets/images/custom-empty-image.png'

const router = useRouter()
const person = userPerson();

const userinfo = $computed(() => person.userinfo);
const address = $computed(() => person.address);

const expandedKeys = $ref([]);
const selectedKeys = $ref([]);
let treeData = $ref([]);

const inviteUrl = $computed(() => `https://${window.location.host}/#/?inviteCode=-inviteTdh-${person.address}-inviteTdh-`)

const copyToClipboard = (text) => {
  copy(text);
  showToast(lang("内容已复制到剪贴板"))
}

const formatAddress = (value) => {
  const frontSix = value.slice(0, 6);
  const backSix = value.slice(-4);
  const middle = '...';
  return frontSix + middle + backSix;
}

const onLoadData = (treeNode) => {
  return new Promise(async (resolve) => {
    if (treeNode.dataRef.children) {
      resolve();
      return;
    }

    const res = await request.get(`app_server/recommend_list?address=${treeNode.dataRef.address}`);

    // res.area = [
    //   {
    //     address: formatAddress("aaaa12378123hjdfowis88883"),
    //     locationId: "3",
    //     countLow: 2,
    //   },
    //   {
    //     address: formatAddress("32423998f8uijkjkrejkw2223"),
    //     locationId: "4",
    //     countLow: 0
    //   }
    // ]
    setTimeout(() => {
      treeNode.dataRef.children = res.recommends.map((item, index) => {
        return {
          title: `${formatAddress(item.address)}---(${lang('数量')}:${item.amount})`,
          key: `${treeNode.eventKey}-${index}`,
          amount: item.amount,
          address: item.address,
          isLeaf: false
        }
      })
      treeData = [...treeData];
      resolve();
    }, 1000);
  });
};

const getUserArea = async () => {
  const res = await request.get(`app_server/recommend_list?address=${address}`);
  treeData = res.recommends.map((item, index) => {
    return {
      title: `${formatAddress(item.address)}---(${lang('数量')}:${item.amount})`,
      key: index,
      address: item.address,
      isLeaf: item.countLow === 0
    }
  })
}

getUserArea()

const depositList = $ref([])
const depositCount = $ref(0)
const depositPage = $ref(1)
const depositAllPageCount = $ref(1)
const depositPageSize = 20

const getTeamDepositList = async (page = 1) => {
  const res = await request.get("app_server/user_team_deposit_list", {
    params: { page }
  })
  depositList = res.list || []
  depositCount = Number(res.count) || 0
  depositAllPageCount = Math.max(1, Math.ceil(depositCount / depositPageSize))
}

getTeamDepositList()

const handleBack = () => {
  router.back()
}

</script>
<style lang='less' scoped>
  .shop-page {
    min-height: 100vh;
    background: url('../../assets/images/topbg7.png') no-repeat;
    background-size: 100% auto;
    .page-main {
      width: 100%;
      padding: 15px 15px 80px 15px;
      box-sizing: border-box;
      .page-title {
        padding: 30px 0;
        h2 {
          font-size: 16px;
        }
        p {
          line-height: 1.6;
        }
      }
      .info-box {
        margin-bottom: 13px;
        width: 100%;
        min-height: 93px;
        background: hsla(0, 0%, 100%, .1);
        border-radius: 18px;
        padding: 15px 15px 20px 15px;
        box-sizing: border-box;
        display: flex;
        flex-direction: column;
        gap: 10px;
        .info-title {
          color: rgb(255, 209, 39);
          font-size: 16px;
        }
        .info-address {
          word-break: break-all;
        }
        .info-link {
          display: flex;
          gap: 20px;
          span {
            word-break: break-all;
          }
          i {
            flex-shrink: 0;
            width: 32px;
            height: 32px;
            background: url('data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACAAAAAgCAYAAABzenr0AAAC6UlEQVR4AexXTWgTQRR+M7tbYpvQWlLw0EMOCh4VLJpGIUcPHhSTEEWQoIWIesjfPVdr2ngpVSz0UolNCz2IUPBgwGztQdCD4MGD3lQaMNJAI4k7vllYTLK7ye5mBQ8O82bn571vvryZeTOh0Ce9zIfF9UIoWV4IfUBhDmVvoxBaLs/PHDGaypQAn3zP194iBJbR8DiK0+xnBJIgjLzeLAR1OKYEvo217gCwC05nNbALKCC8WLt3erpzzJQAJXC9U9GVOmHTI4JQ7MQyJRDLyidjGZk4FQUgSABWOidT64RESouhU2odC1MCODZUjmfk3WhGniMUrvUCiQA3tD6VQHkp7F1fCD3GXX6A4nS3a3b75cLshrbW0ZT8BD2xpk3Iv4yx8/zLha7mwx742XqGSjexw4MybPYCunlEFF9pJH4BLHWDkgA/ZbyPHhprJYFBGNxPAUmQ7nPYg0npHf7ANq9rUh8HL69TSv+sB+9wUwhhEe7hRKLSBCANMEh8D6hMDMbc6BInxqHvsnICbkzkGOM/ASMPbLeZNGU3AoJH8hEGD+2uhY6AorDc1WylZhcodrvS8Deku2iHOx5Li5n26lEGGCl7e621P4N9W6qDFkmxvBg8ulUMT9gR/uAY9bZXEa/vscPxrqwnwKMiox9bSuu7HcEHxxcMPPEudAsNPQELRm6q/JsE+MVBCKnbEoCuy8aql4w8sC1SaSqarh62I41JyUcAHoDNpCNAmZK6lKrUbeJAAm88/76UQ7vh4gBQ6vh2dCUOKAyK5fnZE3ZiANctFc8ERn3tEnrAShz4qnlZtwQIcBYE8tZODOC6giJ8IsAuov2g3BQo488/Vc+IgDrgfsFq+FbcxP8b5y6ndp5r+NiGmtb4G9/6D1A3JT7Rj8XS1WgkLb/pnIfig/RRZ4fL9d1Enr8HzVFpNCuvEEaemqs4Hmng1T43yFrdA9Fs9QpjcAuV8SRhOVxu8rVuM2Umntt5PwjqNwAAAP//ec0etAAAAAZJREFUAwA1x8ykU4MciwAAAABJRU5ErkJggg==') no-repeat;
            background-size: 20px 20px;
          }
        }
      }
      .performance-list {
        min-height: 353px;
        background: hsla(0, 0%, 100%, .1);
        border-radius: 18px;
        padding: 15px;
        background-image: url(../../assets/images/boxbg2.png);
        background-repeat: no-repeat;
        background-size: 100% 218px;
        box-sizing: border-box;
        .performance-info {
          display: flex;
          justify-content: space-between;
          background: hsla(0, 0%, 100%, .05);
          border: 1px solid #444;
          border-radius: 18px;
          margin-bottom: 20px;
          .performance-info-item {
            height: 83px;
            flex: 1 0 0;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            gap: 10px;
          }
        }
        .performance-share-title {
          margin-bottom: 10px;
        }
        .performance-share-list {
          width: 100%;
          display: flex;
          .performance-share-header {
            width: 100%;
            display: flex;
            span {
              flex: 1 0 0;
            }
          }
        }
      }
      .team-deposit-list {
        margin-top: 13px;
        min-height: 200px;
        background: hsla(0, 0%, 100%, .1);
        border-radius: 18px;
        padding: 15px;
        box-sizing: border-box;
        .team-deposit-title {
          margin-bottom: 10px;
          color: rgb(255, 209, 39);
          font-size: 16px;
        }
        .team-deposit-table {
          height: 280px;
          display: flex;
          flex-direction: column;
          overflow: hidden;
          .table-row {
            display: flex;
            align-items: center;
            gap: 8px;
            span {
              font-size: 12px;
              &:nth-child(1) {
                flex: 0 0 108px;
                text-align: left;
              }
              &:nth-child(2) {
                flex: 0 0 44px;
                text-align: center;
              }
              &:nth-child(3) {
                flex: 1;
                min-width: 0;
                text-align: right;
                white-space: nowrap;
              }
            }
          }
          .table-header {
            flex-shrink: 0;
            border-bottom: 0.5px dashed #999;
            padding-bottom: 10px;
            margin-bottom: 5px;
            .table-row span {
              color: #999;
            }
          }
          .table-body {
            flex: 1;
            min-height: 0;
            overflow-y: auto;
            -webkit-overflow-scrolling: touch;
            .table-row {
              padding: 10px 0;
              border-bottom: 0.5px dashed rgba(255, 255, 255, 0.1);
            }
          }
          .empty {
            flex: 1;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            img {
              width: 50px;
              height: 50px;
              opacity: 0.3;
            }
            .empty-text {
              margin-top: 15px;
              font-size: 14px;
              color: rgba(255, 255, 255, 0.3);
            }
          }
        }
        .team-deposit-pagination {
          display: flex;
          flex-direction: row;
          align-items: center;
          justify-content: center;
          gap: 12px;
          .page-size-text {
            flex-shrink: 0;
            font-size: 12px;
            color: rgba(255, 255, 255, 0.5);
            white-space: nowrap;
          }
        }
      }
    }
  }
</style>