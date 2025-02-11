<template>
  <div id="app">
    <header id="header">
      <a v-text="sitename" id="sitename"></a>
      <!-- Search bar -->
      <div class="box">
        <div name="search">
          <input
            v-on:input="searchProduct"
            v-model="searchText"
            type="text"
            class="input"
            name="txt"
          />
        </div>
        <i id="searchIcon" class="fas fa-search"></i>
      </div>
      <!-- Sort By button -->
      <div class="sortContainer">
        <div class="sortRadio">
          <input
            v-model="orderOption"
            type="radio"
            name="sort"
            value="Ascending"
            checked
            @change="sortBy"
          />
          <label for="Ascending">Ascending</label>
          <input
            v-model="orderOption"
            type="radio"
            name="sort"
            value="Decending"
            checked
            @change="sortBy"
          />
          <label for="Decending">Decending</label>
        </div>
        <div class="sortOptions">
          <p>Sort By</p>
          <select id="sort" @change="sortBy" v-model="sortOption">
            <option v-for="option in options" :key="option.id">
              {{ option }}
            </option>
          </select>
        </div>
      </div>
      <!-- checkout button -->
      <!-- To disable cart button if has no lessons in basket -->
      <!-- v-bind:disabled="!cart.length > 0" -->
      <button @click="showCheckout" id="checkout">
        <span v-show="cart.length > 0">{{ cartItemCount }}</span>
        <span class="fa solid fa-cart-plus" style="font-size: 20px"></span>
      </button>
    </header>
    <ProductList :products="products" @addToCart="addToCart" v-if="!checkout" />
    <CheckOut
      :cart="cart"
      @removeItem="removeItem"
      @submitForm="submitForm"
      v-else-if="checkout"
    />
  </div>
</template>

<script>
import ProductList from "./components/product.vue";
import CheckOut from "./components/form.vue";

export default {
  name: "App",
  components: {
    ProductList,
    CheckOut,
  },
  data() {
    return {
      checkout: false,
      sitename: "After School Club",
      products: null,
      cart: [],
      orderOption: "",
      sortOption: "",
      options: ["Price", "Lessons", "Availability", "Location"],
      searchText: "",
      currentID: null,
      orderID: null,
      completedOrder: false,
    };
  },
  // fetching a list of lesson once the app opens
  created() {
    this.getLessonsfromDB();
  },
  methods: {
    getLessonsfromDB() {
      fetch("https://after-school-club.herokuapp.com/collection/lessons")
        .then((res) => res.json())
        .then((data) => {
          this.products = data;
        });
    },
    // Add products to cart
    addToCart(product) {
      this.currentID = product._id;
      console.log(this.currentID);
      const index = this.cart.findIndex(
        (cartItem) => cartItem.product._id === product._id
      );

      if (index >= 0) {
        this.cart[index].quantity += 1;
      } else {
        const cartItem = {
          product: product,
          quantity: 1,
        };
        this.cart.push(cartItem);
      }
      this.updateSpaces(product._id, product.spaces, 1);
    },
    // Toggle checkout
    showCheckout() {
      this.checkout = !this.checkout;
      this.getLessonsfromDB();
      this.completedOrder = false;
    },
    // Checks if there is space to add to cart
    canAddToCart(product) {
      return product.spaces > this.cartCount(product);
    },
    // Counts the products in the cart
    cartCount(product) {
      let count = 0;
      for (let i = 0; i < this.cart.length; i++) {
        if (this.cart[i].id === product.id) {
          count++;
        }
      }
      return count;
    },
    updateSpaces(productId, spaces, quantity) {
      const leftSpace = spaces - quantity;
      fetch(
        `https://after-school-club.herokuapp.com/collection/lessons/${productId}`,
        {
          method: "PUT",
          headers: {
            accept: "application/json, text/plain, */*",
            "content-Type": "application/json",
          },
          body: JSON.stringify({ spaces: leftSpace }),
        }
      )
        .then((res) => res.json())
        .then(() => {
          this.getLessonsfromDB();
        });
    },

    // remove lessons from cart then add back removed
    removeItem(carts) {
      carts.quantity -= 1;
      if (carts.quantity == 0) {
        this.cart = this.cart.filter(
          (item) => item.product._id != carts.product._id
        );
      }
      // Returing spaces after removing from basket
      const currentProduct = this.products.filter(
        (product) => product._id == carts.product._id
      );
      fetch(
        `https://after-school-club.herokuapp.com/collection/lessons/${carts.product._id}`,
        {
          method: "PUT",
          headers: {
            accept: "application/json, text/plain, */*",
            "content-Type": "application/json",
          },
          body: JSON.stringify({ spaces: currentProduct[0].spaces + 1 }),
        }
      )
        .then((res) => res.json())
        .then(() => {
          this.getLessonsfromDB();
        });
    },

    // fetch request to search products in database
    searchProduct() {
      fetch(
        "https://after-school-club.herokuapp.com/collection/lessons/search?q=" +
          this.searchText
      )
        .then((res) => res.json())
        .then((data) => {
          this.products = data;
        });
    },

    async submitForm(fullName, phoneNumber) {
      console.log(fullName);
      console.log(phoneNumber);
      this.cart.forEach((carts) => {
        const {
          product: { _id, spaces, name },
          quantity,
        } = carts;
        fetch("https://after-school-club.herokuapp.com/collection/orders", {
          method: "POST",
          headers: {
            accept: "application/json, text/plain, */*",
            "content-Type": "application/json",
          },
          body: JSON.stringify({
            lessonID: _id,
            subject: name,
            space: quantity,
            name: fullName,
            number: phoneNumber,
            PurchaseDate: new Date().toDateString(),
          }),
        })
          .then((res) => res.json())
          .then((res) => {
            this.orderId = res.orderId;
            this.updateSpaces(_id, spaces, quantity);
            this.cart = [];
          });
      });
    },

    // Sort products by
    sortBy() {
      switch (this.sortOption) {
        case "Price":
          if (this.orderOption == "Ascending") {
            this.products.sort((a, b) => a.price - b.price);
          } else {
            this.products.sort((a, b) => b.price - a.price);
          }
          break;

        case "Availability":
          if (this.orderOption == "Ascending") {
            this.products.sort((a, b) => a.spaces - b.spaces);
          } else {
            this.products.sort((a, b) => b.spaces - a.spaces);
          }
          break;

        case "Location":
          if (this.orderOption == "Ascending") {
            this.products.sort((a, b) =>
              a.location.toLowerCase().localeCompare(b.location.toLowerCase())
            );
          } else {
            this.products.sort((a, b) =>
              b.location.toLowerCase().localeCompare(a.location.toLowerCase())
            );
          }
          break;

        case "Lessons":
          if (this.orderOption == "Ascending") {
            this.products.sort((a, b) =>
              a.name.toLowerCase().localeCompare(b.name.toLowerCase())
            );
          } else {
            this.products.sort((a, b) =>
              b.name.toLowerCase().localeCompare(a.name.toLowerCase())
            );
          }

          break;

        default:
          break;
      }
    },
  },
  computed: {
    cartItemCount() {
      return this.cart.length;
    },
  },
};
</script>

<style>
@import "/src/assets/style.css";
</style>
