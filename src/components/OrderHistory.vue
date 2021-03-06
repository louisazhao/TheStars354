<template>
  <div>
    <b-card-body class="text-left">
      <h2>Your Order History: </h2>
      <b-card v-if="!hasOrderHistroy">
        You have no order History...
        <br/>
        <b-link href="/">Go to shopping</b-link>
      </b-card>
      <b-card v-if="hasOrderHistroy">
        <div class="overflow-auto">
          <b-pagination
            size="sm"
            align="center"
            v-model="currentPage"
            :total-rows="rows"
            :per-page="perPage"
          ></b-pagination>

          <b-card v-for="order in orderList" :key="order.order_id" no-body class="overflow-hidden"  :per-page="perPage" :current-page="currentPage">
            <b-card-body>
                <b-row>
                  <b-col>Order Number: {{order.order_id}}</b-col>
                  <b-col>Order placed: {{order.purchase_date.substring(0,16)}}</b-col>
                </b-row>
                <br/>
                <b-row>
                  <b-col>Shipped to: {{getShippingAddress(order)}}</b-col>
                </b-row>
                <b-row>
                  <b-col>Status: Shipped</b-col>
                </b-row>
                <br/>
                <b-row v-for="item in order.items" :key="item">
                  <b-col>
                    <b-img height="150px" width="150px" :src="item.item.images" class="rounded-0"></b-img>
                  </b-col>
                  <b-col sm="4"><b-link :to="'item-details/' + item.item.item_id">{{item.item.item_name}} (x{{item.order_item_quantity}})</b-link></b-col>
                  <b-col sm="3">${{(item.item.price*(1.0-item.item.discount)).toFixed(2)}}</b-col>
                  <b-col span="1" ><b-link @click="findModal('review', item.item.item_id)">Write/update review</b-link></b-col>
                </b-row>
                <br/>
                <b-row>
                  <b-col>Subtotal: {{subtotalTxt(order.items)}}</b-col>
                </b-row>
                <b-row v-if="hasCouponDiscount(order)">
                  <b-col>Discount: {{discountTxt(order)}}</b-col>
                </b-row>
                <b-row>
                  <b-col>Tax: {{taxTxt(order)}}</b-col>
                </b-row>
                <b-row>
                  <b-col>Total: {{totalTxt(order)}}</b-col>
                </b-row>
            </b-card-body>
           </b-card>
        </div>
      </b-card>
    </b-card-body>
    <b-modal ref="review" hide-footer title="Review">
      <b-form-textarea
      id="textarea"
      v-model="reviewInput"
      placeholder="Enter your review here..."
      rows="3"
      max-rows="6"
      ></b-form-textarea>
      <b-form-group label="Rating">
        <b-form-radio-group>
        <b-form-radio v-model="ratingInput" name="ratingInput" value="1">1</b-form-radio>
        <b-form-radio v-model="ratingInput" name="ratingInput" value="2">2</b-form-radio>
        <b-form-radio v-model="ratingInput" name="ratingInput" value="3">3</b-form-radio>
        <b-form-radio v-model="ratingInput" name="ratingInput" value="4">4</b-form-radio>
        <b-form-radio v-model="ratingInput" name="ratingInput" value="5">5</b-form-radio>
        </b-form-radio-group>
      </b-form-group>
      <b-form-group
              label="Images"
      >
          <b-form-file
                  v-for="(image, index) in imagesInput"
                  :key="index"
                  v-model="imagesInput[index]"
                  accept=".jpg, .png, .gif"
                  placeholder="Choose a file or drop it here..."
                  drop-placeholder="Drop file here..."
                  type="file"
                  style="margin-top: 5px;"
          ></b-form-file>
      </b-form-group>
      <br/>
      <b-button type="submit" variant="outline-success" @click.prevent="addReview" block>Add Review</b-button>
    </b-modal>
  </div>
</template>

<script>
import axios from 'axios'

export default {
  data () {
    return {
      imagesInput: [null, null, null, null, null],
      modal_item_id: -1,
      hasOrderHistroy: false,
      perPage: 5,
      currentPage: 1,
      reviewInput: '',
      ratingInput: '',
      orders: [],
      address1: '',
      address2: '',
      address3: ''
    }
  },
  mounted () {
    var url = 'api/resource/buyerInfo?uid=' + encodeURIComponent(this.$store.state.uid)
    axios
      .get(url)
      .then(response => {
        this.orders = response.data['order_history']
        if (this.orders.length !== 0) {
          this.hasOrderHistroy = true
        }
        this.address1 = response.data['address1']
        this.address2 = response.data['address2']
        this.address3 = response.data['address3']
      })
      .catch(error => alert(error))
  },
  computed: {
    rows () {
      return this.orders.length
    },
    orderList () {
      return this.orders.slice((this.currentPage - 1) * this.perPage, this.currentPage * this.perPage)
    }
  },
  methods: {
    findModal (modal, id) {
      this.reviewInput = ''
      this.ratingInput = ''
      this.modal_item_id = id
      this.$refs[modal].show()
    },
    addReview () {
      var formData = new FormData()
      for (var i = 0; i < 5; i++) {
        formData.append('image' + (i + 1), this.imagesInput[i])
      }
      var url = 'api/resource/review/' + this.modal_item_id + '?content=' + encodeURIComponent(this.reviewInput) + '&rating=' + encodeURIComponent(this.ratingInput)
      axios
        .post(url, formData, { headers: { 'Content-Type': 'multipart/form-data' } })
        .then(response => {
        })
        .catch(error => alert(error))
      this.$refs['review'].hide()
    },
    subtotal (orderItems) {
      var subtotal = 0
      for (var data of orderItems) {
        let price = data.item.price * (1.0 - data.item.discount)
        subtotal += price * data.order_item_quantity
      }
      return subtotal
    },
    subtotalTxt (orderItems) {
      return '$' + this.subtotal(orderItems).toFixed(2)
    },
    hasCouponDiscount (order) {
      return order.coupon_discount !== 0.0
    },
    discountTxt (order) {
      return '$ -' + (this.subtotal(order.items) * order.coupon_discount).toFixed(2)
    },
    taxTxt (order) {
      return '$' + (this.subtotal(order.items) * (1.0 - order.coupon_discount) * 0.15).toFixed(2)
    },
    totalTxt (order) {
      return '$' + (this.subtotal(order.items) * (1.0 - order.coupon_discount) * 1.15).toFixed(2)
    },
    getShippingAddress (order) {
      var addressIndex = order.buyer_address_index
      if (addressIndex === 1) {
        return this.address1
      }
      if (addressIndex === 2) {
        return this.address2
      }
      if (addressIndex === 3) {
        return this.address3
      }
    }
  }
}
</script>
