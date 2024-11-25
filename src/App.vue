<template>
  <div id="app">
    <header>
      <h1>Lessons Shop</h1>
      <div class="cart">
        <span>🛒 Cart: {{ cartItemCount }} items</span>
      </div>
    </header>

    <!-- Search bar and sort options -->
    <div class="controls">
      <input 
        type="text" 
        v-model="searchQuery" 
        placeholder="Search lessons..." 
        class="search-bar"
      />
      <select v-model="sortOption" class="sort-select">
        <option disabled value="">Sort by</option>
        <option value="name">Name (A-Z)</option>
        <option value="priceAsc">Price (Low to High)</option>
      </select>
    </div>

    <!-- Product List -->
    <div v-if="showProductList" class="product-list">
      <h2>Products</h2>
      <div v-for="product in sortedAndFilteredProducts" :key="product._id" class="product">
        <h3>{{ product.subject }}</h3>
        <img :src="product.imagePath" :alt="product.imageAlt" class="product-image" />
        <p>Location: {{ product.location }}</p>
        <p>Price: ${{ product.price }}</p>
        <p>Available: {{ product.stock }} <span v-if="product.stock <= 2" class="low-stock">Low stock!</span></p>
        <div class="cart-buttons">
          <button :disabled="product.stock === 0" @click="addToCart(product)">
            Add to Cart
          </button>
          <!-- Disable Remove button until the product is added to the cart -->
          <button :disabled="!isInCart(product)" @click="removeFromCart(product, 1)">
            Remove
          </button>
        </div>
      </div>

      <button @click="goToCheckout" :disabled="cartItemCount === 0" class="checkout-btn">
        Go to Checkout
      </button>
    </div>

    <!-- Checkout Area -->
    <div v-if="showCheckoutPopup && !showUserForm" class="checkout-area">
      <h2>Checkout</h2>
      <div v-for="(item, index) in cart" :key="index" class="cart-item">
        <h3>{{ item.subject }}</h3>
        <img :src="item.imagePath" :alt="item.imageAlt" class="cart-image" />
        <p><strong>Price:</strong> ${{ item.price }}</p>
        <p><strong>Quantity:</strong> 
          <input type="number" v-model.number="item.quantity" min="1" :max="item.stock" @change="updateCart(index)" />
        </p>
        <p><strong>Subtotal:</strong> ${{ (item.price * item.quantity).toFixed(2) }}</p>
        <div class="cart-buttons">
          <button @click="removeFromCart(item, item.quantity)">Remove</button>
        </div>
      </div>

      <p><strong>Total Price:</strong> ${{ totalPrice.toFixed(2) }}</p>
      <button @click="goBackToProducts" class="back-btn">Back to Products</button>
      <button @click="showUserFormPopup" class="finalize-btn">
        Finalize Order
      </button>
    </div>

    <!-- User Form Pop-up -->
    <div v-if="showUserForm" class="user-form-overlay">
      <div class="user-form-container">
        <h2>Enter Your Details</h2>
        <form @submit.prevent="finalizeOrder">
          <label for="name">Name</label>
          <input type="text" id="name" v-model="user.name" required />

          <label for="email">Email</label>
          <input type="email" id="email" v-model="user.email" required />

          <label for="address">Address</label>
          <input type="text" id="address" v-model="user.address" required />

          <button type="submit">Submit Order</button>
        </form>
        <button @click="closeUserFormPopup" class="back-btn">Back to Cart</button>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      products: [],
      cart: [],
      user: { name: '', email: '', address: '' },
      searchQuery: '',
      sortOption: '',
      showCheckoutPopup: false,
      showProductList: true,
      showUserForm: false // Show the user form as a pop-up
    };
  },
  created() {
    this.fetchProducts();
  },
  methods: {
    async fetchProducts() {
      try {
        const response = await fetch('http://localhost:5000/api/lessons');
        const products = await response.json();
        this.products = products;
      } catch (error) {
        console.error('Error fetching products:', error);
      }
    },
    addToCart(product) {
      const existing = this.cart.find(item => item._id === product._id);
      if (existing) {
        existing.quantity++;
      } else {
        this.cart.push({ ...product, quantity: 1 });
      }
      product.stock--;
    },
    async removeFromCart(product, quantity) {
      const index = this.cart.findIndex(item => item._id === product._id);
      if (index >= 0) {
        const cartItem = this.cart[index];
        if (cartItem.quantity > quantity) {
          cartItem.quantity -= quantity;
        } else {
          this.cart.splice(index, 1);
        }
        product.stock += quantity;

        try {
          await fetch('http://localhost:5000/api/cart/remove', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ lessonId: product._id, quantity })
          });
        } catch (error) {
          console.error('Error updating stock:', error);
        }
      }
    },
    goToCheckout() {
      this.showProductList = false;
      this.showCheckoutPopup = true;
    },
    goBackToProducts() {
      this.showProductList = true;
      this.showCheckoutPopup = false;
    },
    showUserFormPopup() {
      this.showUserForm = true; // Show the user form in a pop-up
    },
    closeUserFormPopup() {
      this.showUserForm = false; // Close the user form pop-up and go back to the cart
    },
    async finalizeOrder() {
      try {
        const order = {
          user: { ...this.user },
          cart: this.cart,
          totalPrice: this.totalPrice
        };

        const response = await fetch('http://localhost:5000/api/orders', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(order)
        });

        if (response.ok) {
          await this.updateStock();
          this.cart = [];
          this.user = { name: '', email: '', address: '' };
          this.showUserForm = false;
          this.showCheckoutPopup = false;
          this.showProductList = true;
          alert('Order placed successfully!');
        }
      } catch (error) {
        console.error('Error finalizing order:', error);
      }
    },
    async updateStock() {
      for (const item of this.cart) {
        try {
          await fetch(`http://localhost:5000/api/lessons/${item._id}/update-stock`, {
            method: 'PATCH',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ quantity: item.quantity })
          });
        } catch (error) {
          console.error(`Error updating stock for ${item.subject}:`, error);
        }
      }
      await this.fetchProducts();
    },
    isInCart(product) {
      return this.cart.some(item => item._id === product._id);
    }
  },
  computed: {
    cartItemCount() {
      return this.cart.reduce((sum, item) => sum + item.quantity, 0);
    },
    totalPrice() {
      return this.cart.reduce((sum, item) => sum + item.price * item.quantity, 0);
    },
    sortedAndFilteredProducts() {
      let filtered = this.products.filter(p => p.subject.toLowerCase().includes(this.searchQuery.toLowerCase()));
      
      if (this.sortOption) {
        switch (this.sortOption) {
          case 'subject':
            filtered = filtered.sort((a, b) => a.subject.localeCompare(b.subject));
            break;
          case 'location':
            filtered = filtered.sort((a, b) => a.location.localeCompare(b.location));
            break;
          case 'priceAsc':
            filtered = filtered.sort((a, b) => a.price - b.price);
            break;
          case 'stockAsc':
            filtered = filtered.sort((a, b) => a.stock - b.stock);
            break;
          default:
            break;
        }
      }
      
      return filtered;
    }
  }
};
</script>

<style scoped>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  text-align: center;
}

.product-list {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
}

.product {
  border: 1px solid #ccc;
  padding: 10px;
  margin: 10px;
  width: 300px;
  text-align: center;
}

.product-image {
  width: 100px;
  height: 100px;
  object-fit: cover;
}

.low-stock {
  color: red;
  font-weight: bold;
}

.cart-buttons {
  display: flex;
  justify-content: space-between;
}

button {
  background-color: #4CAF50;
  color: white;
  padding: 10px 15px;
  border: none;
  cursor: pointer;
}

button:disabled {
  background-color: #ddd;
}

.checkout-btn {
  margin-top: 20px;
}

.checkout-area {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.cart-item {
  margin-bottom: 20px;
}

.cart-image {
  width: 80px;
  height: 80px;
}

.finalize-btn, .back-btn {
  margin-top: 20px;
}

.user-form-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
}

.user-form-container {
  background: white;
  padding: 30px;
  border-radius: 8px;
  width: 300px;
}

input[type="text"], input[type="email"], input[type="number"] {
  padding: 10px;
  margin: 10px;
  width: 100%;
  border: 1px solid #ccc;
}
</style>
