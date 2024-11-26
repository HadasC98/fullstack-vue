<template>
  <div id="app">
    <header>
      <h1>Lessons Shop</h1>
      <div class="cart">
        <span>🛒 Cart: {{ cartItemCount }} items</span>
      </div>
    </header>

    <div class="controls">
      <input 
        type="text" 
        v-model="searchQuery" 
        placeholder="Search lessons..." 
        class="search-bar"
      />
      <select v-model="sortOption" class="sort-select">
        <option disabled value="">Sort by</option>
        <option value="subject">Subject (A-Z)</option>
        <option value="location">Location (A-Z)</option>
        <option value="priceAsc">Price (Low to High)</option>
        <option value="stockAsc">Availability (Low to High)</option>
      </select>
    </div>

    <div v-if="products.length === 0" class="no-products">
      <p>No lessons available at the moment.</p>
    </div>

    <div v-if="showProductList" class="product-list">
      <h2>Products</h2>
      <div v-for="product in sortedAndFilteredProducts" :key="product._id" class="product">
        <h3>{{ product.subject }}</h3>
        <img :src="product.imagePath" :alt="product.imageAlt" class="product-image" />
        <p>Location: {{ product.location }}</p>
        <p>Price: ${{ product.price }}</p>
        <p>Available: {{ product.stock }} 
          <span v-if="product.stock <= 2" class="low-stock">Low stock!</span>
        </p>
        <div class="cart-buttons">
          <button :disabled="product.stock === 0" @click="addToCart(product)">
            Add to Cart
          </button>
          <button :disabled="!isInCart(product)" @click="removeFromCart(product)">
            Remove from Cart
          </button>
        </div>
      </div>

      <button @click="goToCheckout" :disabled="cartItemCount === 0" class="checkout-btn">
        Go to Checkout
      </button>
    </div>

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

    <div v-if="showUserForm" class="user-form-overlay">
      <div class="user-form-container">
        <h2>Enter Your Details</h2>
        <form @submit.prevent="finalizeOrder">
          <label for="name">Name</label>
          <input type="text" id="name" v-model="user.name" required />
          <label for="phone">Phone</label>
          <input type="text" id="phone" v-model="user.phone" required />
          <label for="email">Email</label>
          <input type="email" id="email" v-model="user.email" required />
          <label for="address">Address</label>
          <input type="text" id="address" v-model="user.address" required />
          <button type="submit" :disabled="!isFormValid">Submit Order</button>
          <button @click="closeUserFormPopup">Cancel</button>
        </form>
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
      user: { name: '', phone: '', email: '', address: '' },
      searchQuery: '',
      sortOption: '',
      showCheckoutPopup: false,
      showProductList: true,
      showUserForm: false
    };
  },
  mounted() {
    this.fetchProducts();
  },
  methods: {
    async fetchProducts() {
  try {
    const response = await fetch('https://fullstack-express-9dbh.onrender.com/api/lessons');
    if (!response.ok) {
      throw new Error('Failed to fetch lessons');
    }
    const data = await response.json();
    console.log('Lessons fetched:', data); // Log data to check
    this.products = data;
  } catch (error) {
    console.error('Error fetching products:', error);
    alert('Error fetching products');
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
      this.updateProductStock(product);
    },
    async updateProductStock(product) {
      try {
        await fetch(`https://fullstack-express-9dbh.onrender.com/api/lessons/${product._id}/update-stock`, {
          method: 'PATCH',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ quantity: -1 })
        });
      } catch (error) {
        console.error(`Error updating stock for ${product.subject}:`, error);
      }
    },
    removeFromCart(product, quantity = 1) {
      const index = this.cart.findIndex(item => item._id === product._id);
      if (index >= 0) {
        const cartItem = this.cart[index];
        if (cartItem.quantity > quantity) {
          cartItem.quantity -= quantity;
        } else {
          this.cart.splice(index, 1);
        }
        product.stock += quantity;
        this.updateProductStock(product);
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
      this.showUserForm = true;
    },
    closeUserFormPopup() {
      this.showUserForm = false;
    },
    async finalizeOrder() {
      const lessonIDs = this.cart.map(item => item._id);
      const numberOfSpaces = this.cart.map(item => item.quantity);
      const order = {
        name: this.user.name,
        phoneNumber: this.user.phone,
        email: this.user.email,
        address: this.user.address,
        lessonIDs,
        numberOfSpaces
      };

      try {
        const response = await fetch('https://fullstack-express-9dbh.onrender.com/api/orders', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(order)
        });

        if (response.ok) {
          // Save the user to the users collection
          await fetch('https://fullstack-express-9dbh.onrender.com/api/users', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ name: this.user.name, phoneNumber: this.user.phone })
          });
          
          // Clear cart and refresh products
          this.cart = [];
          this.user = { name: '', phone: '', email: '', address: '' };
          this.fetchProducts();
          alert('Order placed successfully!');
        } else {
          alert('Failed to place the order');
        }
      } catch (error) {
        console.error('Error finalizing order:', error);
      }
    },
    isInCart(product) {
      return this.cart.some(item => item._id === product._id);
    },
    isFormValid() {
      const nameRegex = /^[A-Za-z]+$/;
      const phoneRegex = /^[0-9]+$/;
      return nameRegex.test(this.user.name) && phoneRegex.test(this.user.phone);
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
      let filtered = this.products.filter(product =>
        product.subject.toLowerCase().includes(this.searchQuery.toLowerCase())
      );
      switch (this.sortOption) {
        case 'subject':
          filtered.sort((a, b) => a.subject.localeCompare(b.subject));
          break;
        case 'location':
          filtered.sort((a, b) => a.location.localeCompare(b.location));
          break;
        case 'priceAsc':
          filtered.sort((a, b) => a.price - b.price);
          break;
        case 'stockAsc':
          filtered.sort((a, b) => a.stock - b.stock);
          break;
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


