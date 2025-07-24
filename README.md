<!DOCTYPE html>
<html lang="vi">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Hệ Thống Quản Lý Kinh Doanh</title>
    <!-- Tailwind CSS CDN -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Google Fonts: Inter and Roboto -->
    <link
      href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&family=Roboto:wght@300;400;500;700&display=swap"
      rel="stylesheet"
    />
    <!-- Font Awesome for Icons -->
    <link
      rel="stylesheet"
      href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0-beta3/css/all.min.css"
      xintegrity="sha512-Fo3rlrZj/k7ujTnHg4CGR2D7kSs0v4LLanw2qksYuRlEzO+tcaEPQogQ0KaoGN26/zrn20ImR1DfuLWnOo7aBA=="
      crossorigin="anonymous"
      referrerpolicy="no-referrer"
    />
    <!-- Chart.js CDN for charts -->
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <!-- Firebase SDKs -->
    <script type="module">
      import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
      import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
      import { getFirestore, collection, addDoc, getDocs, onSnapshot, doc, updateDoc, deleteDoc, query, where, getDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

      // Global variables for Firebase instances and user data
      window.firebaseApp = null;
      window.db = null;
      window.auth = null;
      window.userId = null;
      window.isAuthReady = false;

      // Global data stores (will be populated from Firestore)
      window.products = [];
      window.suppliers = [];
      window.units = [{id: 'default-unit-1', name: 'Cái'}, {id: 'default-unit-2', name: 'Kg'}, {id: 'default-unit-3', name: 'Mét'}]; // Default units
      window.categories = [{id: 'default-cat-1', name: 'Vật tư'}, {id: 'default-cat-2', name: 'Hàng hóa'}, {id: 'default-cat-3', name: 'Dịch vụ'}]; // Default categories
      window.banks = [{id: 'default-bank-1', name: 'Vietcombank'}, {id: 'default-bank-2', name: 'Techcombank'}]; // Default banks
      window.entities = [{id: 'default-entity-1', name: 'Kho tổng', type: 'both'}]; // Default entity for sender/recipient
      window.orders = [];
      window.debtRecords = [];
      window.inventoryMovements = [];

      // Get the app ID and Firebase config from the global variables (provided by Canvas environment)
      const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
      const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
      const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

      // Initialize Firebase and handle authentication
      const initializeFirebase = async () => {
          try {
              window.showLoadingOverlay();
              window.firebaseApp = initializeApp(firebaseConfig);
              window.db = getFirestore(window.firebaseApp);
              window.auth = getAuth(window.firebaseApp);

              onAuthStateChanged(window.auth, async (user) => {
                  if (user) {
                      window.userId = user.uid;
                  } else {
                      if (!initialAuthToken) {
                          await signInAnonymously(window.auth);
                          window.userId = window.auth.currentUser?.uid || crypto.randomUUID(); // Fallback for anonymous
                      }
                  }
                  window.isAuthReady = true;
                  window.hideLoadingOverlay();
                  console.log("Firebase initialized. User ID:", window.userId);
                  // Start listening to data once authenticated
                  setupFirestoreListeners();
              });

              if (initialAuthToken) {
                  await signInWithCustomToken(window.auth, initialAuthToken);
              }

          } catch (error) {
              console.error("Error initializing Firebase or signing in:", error);
              window.showMessage('error', `Lỗi khi khởi tạo Firebase: ${error.message}`);
              window.hideLoadingOverlay();
          }
      };

      // Setup real-time listeners for all collections
      const setupFirestoreListeners = () => {
          if (!window.db || !window.userId) {
              console.warn("Firestore or User ID not ready for listeners.");
              return;
          }

          // Products
          onSnapshot(collection(window.db, `artifacts/${appId}/users/${window.userId}/products`), (snapshot) => {
              window.products = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
              window.products.sort((a, b) => (a.name || '').localeCompare(b.name || ''));
              console.log("Products updated:", window.products);
              window.renderProductList(); // Re-render product list
              window.populateProductDropdowns(); // Update product dropdowns
          }, (error) => console.error("Error fetching products:", error));

          // Suppliers
          onSnapshot(collection(window.db, `artifacts/${appId}/users/${window.userId}/suppliers`), (snapshot) => {
              window.suppliers = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
              window.suppliers.sort((a, b) => (a.name || '').localeCompare(b.name || ''));
              console.log("Suppliers updated:", window.suppliers);
              window.renderSupplierList(); // Re-render supplier list
              window.populateSupplierDropdowns(); // Update supplier dropdowns
          }, (error) => console.error("Error fetching suppliers:", error));

          // Units
          onSnapshot(collection(window.db, `artifacts/${appId}/users/${window.userId}/units`), (snapshot) => {
              window.units = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
              window.units.sort((a, b) => (a.name || '').localeCompare(b.name || ''));
              console.log("Units updated:", window.units);
              window.populateUnitDropdowns(); // Update unit dropdowns
          }, (error) => console.error("Error fetching units:", error));

          // Categories
          onSnapshot(collection(window.db, `artifacts/${appId}/users/${window.userId}/categories`), (snapshot) => {
              window.categories = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
              window.categories.sort((a, b) => (a.name || '').localeCompare(b.name || ''));
              console.log("Categories updated:", window.categories);
              window.populateCategoryDropdowns(); // Update category dropdowns
          }, (error) => console.error("Error fetching categories:", error));

          // Banks
          onSnapshot(collection(window.db, `artifacts/${appId}/users/${window.userId}/banks`), (snapshot) => {
              window.banks = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
              window.banks.sort((a, b) => (a.name || '').localeCompare(b.name || ''));
              console.log("Banks updated:", window.banks);
              window.populateBankDropdowns(); // Update bank dropdowns
              window.populatePaymentMethodDropdowns(); // Update payment methods with new banks
          }, (error) => console.error("Error fetching banks:", error));

          // Entities (for sender/recipient in inventory)
          onSnapshot(collection(window.db, `artifacts/${appId}/users/${window.userId}/entities`), (snapshot) => {
              window.entities = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
              window.entities.sort((a, b) => (a.name || '').localeCompare(b.name || ''));
              console.log("Entities updated:", window.entities);
              window.populateEntityDropdowns(); // Update entity dropdowns
          }, (error) => console.error("Error fetching entities:", error));

          // Orders (Lịch sử đặt hàng)
          onSnapshot(collection(window.db, `artifacts/${appId}/users/${window.userId}/orders`), (snapshot) => {
              window.orders = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
              window.orders.sort((a, b) => new Date(b.orderDate) - new Date(a.orderDate)); // Sort by date descending
              console.log("Orders updated:", window.orders);
              window.renderOrderHistory(); // Re-render order history
              window.renderDashboardCharts(); // Update dashboard charts with new order data
          }, (error) => console.error("Error fetching orders:", error));

          // Debt Records (Công nợ)
          onSnapshot(collection(window.db, `artifacts/${appId}/users/${window.userId}/debtRecords`), (snapshot) => {
              window.debtRecords = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
              window.debtRecords.sort((a, b) => new Date(a.dueDate) - new Date(b.dueDate)); // Sort by due date
              console.log("Debt records updated:", window.debtRecords);
              window.renderDebtList(); // Re-render debt list
              window.renderDebtCharts(); // Update debt charts
          }, (error) => console.error("Error fetching debt records:", error));

          // Inventory Movements (Xuất/Nhập/Bàn giao)
          onSnapshot(collection(window.db, `artifacts/${appId}/users/${window.userId}/inventoryMovements`), (snapshot) => {
              window.inventoryMovements = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
              window.inventoryMovements.sort((a, b) => new Date(b.date) - new Date(a.date)); // Sort by date descending
              console.log("Inventory movements updated:", window.inventoryMovements);
              window.renderInventoryList(); // Re-render inventory list
          }, (error) => console.error("Error fetching inventory movements:", error));
      };

      // Call initializeFirebase when the window loads
      window.onload = initializeFirebase;

      // Message Box Functions (Replacing alert/confirm)
      window.showMessage = (type, message) => {
          const msgBox = document.getElementById('messageContainer');
          if (!msgBox) {
              console.error("Message container not found.");
              return;
          }
          msgBox.className = `message-box show ${type}`;
          msgBox.innerHTML = `<span>${message}</span><button onclick="window.hideMessage()">X</button>`;
          setTimeout(() => {
              window.hideMessage();
          }, 5000); // Hide after 5 seconds
      };

      window.hideMessage = () => {
          const msgBox = document.getElementById('messageContainer');
          if (msgBox) {
              msgBox.classList.remove('show');
          }
      };

      // Custom Confirmation Modal
      window.showConfirmation = (message, onConfirm) => {
          const modalHtml = `
              <div id="confirmationModal" class="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-[7000]">
                  <div class="bg-white p-6 rounded-lg shadow-xl w-full max-w-sm">
                      <h3 class="text-xl font-semibold mb-4 text-gray-700">Xác nhận</h3>
                      <p class="mb-6 text-gray-600">${message}</p>
                      <div class="flex justify-end gap-4">
                          <button type="button" class="btn btn-secondary" onclick="window.closeConfirmationModal()">Hủy</button>
                          <button type="button" class="btn btn-primary" id="confirmActionButton">Xác nhận</button>
                      </div>
                  </div>
              </div>
          `;
          document.body.insertAdjacentHTML('beforeend', modalHtml);
          const confirmBtn = document.getElementById('confirmActionButton');
          if (confirmBtn) {
              confirmBtn.onclick = () => {
                  onConfirm();
                  window.closeConfirmationModal();
              };
          }
      };

      window.closeConfirmationModal = () => {
          const modal = document.getElementById('confirmationModal');
          if (modal) {
              modal.remove();
          }
      };

      // Loading Overlay Functions
      window.showLoadingOverlay = () => {
          const loadingOverlay = document.getElementById('loadingOverlay');
          if (loadingOverlay) {
              loadingOverlay.classList.add('visible');
          }
      };

      window.hideLoadingOverlay = () => {
          const loadingOverlay = document.getElementById('loadingOverlay');
          if (loadingOverlay) {
              loadingOverlay.classList.remove('visible');
          }
      };

      // --- Tab Switching Logic ---
      document.addEventListener('DOMContentLoaded', () => {
          const sidebarButtons = document.querySelectorAll('.sidebar-button');
          const tabContents = document.querySelectorAll('.tab-content');

          sidebarButtons.forEach(button => {
              button.addEventListener('click', () => {
                  // Remove active class from all buttons and hide all tab contents
                  sidebarButtons.forEach(btn => btn.classList.remove('active'));
                  tabContents.forEach(content => content.classList.remove('active'));

                  // Add active class to clicked button and show corresponding tab content
                  button.classList.add('active');
                  const targetTabId = button.dataset.tab;
                  const targetTab = document.getElementById(targetTabId);
                  if (targetTab) {
                      targetTab.classList.add('active');
                  }


                  // Re-render charts/data for the activated tab if needed
                  if (targetTabId === 'dashboard') {
                      window.renderDashboardCharts();
                  } else if (targetTabId === 'priceComparison') {
                      window.renderPriceComparison();
                  } else if (targetTabId === 'inventory') {
                      window.renderInventoryList();
                      window.clearInventoryMovementForm(); // Re-initialize form for multiple items
                  } else if (targetTabId === 'debtManagement') {
                      window.renderDebtList();
                      window.renderDebtCharts();
                  } else if (targetTabId === 'supplierInfo') {
                      window.renderSupplierList();
                  } else if (targetTabId === 'product') {
                      window.renderProductList();
                  } else if (targetTabId === 'orderHistory') {
                      window.renderOrderHistory();
                      window.clearOrderForm(); // Re-initialize form for multiple items
                  } else if (targetTabId === 'settings') {
                      window.renderSettingsTab();
                  }
              });
          });

          // Initial render of the active tab (Dashboard by default)
          const dashboardTab = document.getElementById('dashboard');
          if (dashboardTab) {
              dashboardTab.classList.add('active');
          }
      });

      // --- Autocomplete/Dropdown with Add (+) Button Logic ---
      // This function creates an input field with autocomplete and a plus button
      // It's designed to be called dynamically for fields like product name, supplier name, unit, category, bank
      window.createAutocompleteInput = (containerId, inputId, placeholder, dataList, onSelectCallback, onAddCallback) => {
          const container = document.getElementById(containerId);
          if (!container) {
              console.error(`Container with ID ${containerId} not found.`);
              return;
          }

          container.innerHTML = `
              <div class="flex items-center relative autocomplete-container">
                  <input
                      type="text"
                      id="${inputId}"
                      placeholder="${placeholder}"
                      class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
                  />
                  <button type="button" class="plus-button" id="add-${inputId}-btn">+</button>
                  <div id="${inputId}-suggestions" class="autocomplete-suggestions hidden"></div>
              </div>
          `;

          const inputElement = document.getElementById(inputId);
          const addButton = document.getElementById(`add-${inputId}-btn`);
          const suggestionsDiv = document.getElementById(`${inputId}-suggestions`);

          let currentSuggestions = [];

          if (inputElement) {
              inputElement.addEventListener('input', () => {
                  const inputValue = inputElement.value.toLowerCase();
                  suggestionsDiv.innerHTML = '';
                  if (inputValue.length > 0) {
                      currentSuggestions = dataList.filter(item => item.name.toLowerCase().includes(inputValue));
                      if (currentSuggestions.length > 0) {
                          currentSuggestions.forEach(item => {
                              const div = document.createElement('div');
                              div.textContent = item.name;
                              div.dataset.id = item.id;
                              div.addEventListener('click', () => {
                                  inputElement.value = item.name;
                                  onSelectCallback(item.id, item.name);
                                  suggestionsDiv.classList.add('hidden');
                              });
                              suggestionsDiv.appendChild(div);
                          });
                          suggestionsDiv.classList.remove('hidden');
                      } else {
                          suggestionsDiv.classList.add('hidden');
                      }
                  } else {
                      suggestionsDiv.classList.add('hidden');
                  }
              });
          }


          // Hide suggestions when clicking outside
          document.addEventListener('click', (event) => {
              if (!container.contains(event.target)) {
                  if (suggestionsDiv) {
                      suggestionsDiv.classList.add('hidden');
                  }
              }
          });

          // Handle add button click
          if (addButton) {
              addButton.addEventListener('click', onAddCallback);
          }
      };

      // --- Modals for Quick Add ---
      window.openQuickAddModal = (type, currentInputId) => {
          let title = '';
          let label = '';
          let placeholder = '';
          let collectionName = '';
          let inputType = 'text';
          let entityTypeField = ''; // For entities collection

          switch (type) {
              case 'unit':
                  title = 'Thêm Đơn vị tính mới';
                  label = 'Tên Đơn vị tính:';
                  placeholder = 'VD: Cái, Kg, Mét';
                  collectionName = 'units';
                  break;
              case 'category':
                  title = 'Thêm Phân loại mới';
                  label = 'Tên Phân loại:';
                  placeholder = 'VD: Vật tư, Hàng hóa, Dịch vụ';
                  collectionName = 'categories';
                  break;
              case 'bank':
                  title = 'Thêm Ngân hàng mới';
                  label = 'Tên Ngân hàng:';
                  placeholder = 'VD: Vietcombank, Techcombank';
                  collectionName = 'banks';
                  break;
              case 'product-quick':
                  title = 'Thêm Sản phẩm mới';
                  label = 'Tên Sản phẩm:';
                  placeholder = 'Tên sản phẩm';
                  collectionName = 'products';
                  break;
              case 'supplier-quick':
                  title = 'Thêm Nhà cung cấp mới';
                  label = 'Tên Nhà cung cấp:';
                  placeholder = 'Tên nhà cung cấp';
                  collectionName = 'suppliers';
                  break;
              case 'entity-sender':
                  title = 'Thêm Người gửi mới';
                  label = 'Tên Người gửi:';
                  placeholder = 'VD: Kho tổng, Cửa hàng A';
                  collectionName = 'entities';
                  entityTypeField = 'sender';
                  break;
              case 'entity-recipient':
                  title = 'Thêm Đơn vị nhận hàng mới';
                  label = 'Tên Đơn vị nhận hàng:';
                  placeholder = 'VD: Cửa hàng B, Khách hàng C';
                  collectionName = 'entities';
                  entityTypeField = 'recipient';
                  break;
              default:
                  return;
          }

          const modalHtml = `
              <div id="quickAddModal" class="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-[7000]">
                  <div class="bg-white p-6 rounded-lg shadow-xl w-full max-w-md">
                      <h3 class="text-xl font-semibold mb-4 text-gray-700">${title}</h3>
                      <div class="form-group mb-4">
                          <label for="quickAddInput">${label}</label>
                          <input type="${inputType}" id="quickAddInput" placeholder="${placeholder}" class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500" required>
                      </div>
                      ${type === 'product-quick' ? `
                          <div class="form-group mb-4">
                              <label for="quickAddProductPrice">Giá:</label>
                              <input type="number" id="quickAddProductPrice" placeholder="Giá sản phẩm" class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500">
                          </div>
                          <div class="form-group mb-4">
                              <label for="quickAddProductUnit">Đơn vị tính:</label>
                              <select id="quickAddProductUnit" class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white">
                                  <option value="">Chọn ĐVT</option>
                                  ${window.units.map(u => `<option value="${u.id}">${u.name}</option>`).join('')}
                              </select>
                          </div>
                      ` : ''}
                      ${type === 'supplier-quick' ? `
                          <div class="form-group mb-4">
                              <label for="quickAddSupplierContact">Thông tin liên hệ:</label>
                              <input type="text" id="quickAddSupplierContact" placeholder="Email/Điện thoại" class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500">
                          </div>
                      ` : ''}
                      <div class="flex justify-end gap-4">
                          <button type="button" class="btn btn-secondary" onclick="window.closeQuickAddModal()">Hủy</button>
                          <button type="button" class="btn btn-primary" id="saveQuickAddBtn">Lưu</button>
                      </div>
                  </div>
              </div>
          `;
          document.body.insertAdjacentHTML('beforeend', modalHtml);

          const saveQuickAddBtn = document.getElementById('saveQuickAddBtn');
          if (saveQuickAddBtn) {
              saveQuickAddBtn.onclick = async () => {
                  const quickAddInput = document.getElementById('quickAddInput');
                  const value = quickAddInput ? quickAddInput.value.trim() : '';
                  if (!value) {
                      window.showMessage('error', `Vui lòng nhập ${label.toLowerCase().replace(':', '')}`);
                      return;
                  }

                  window.showLoadingOverlay();
                  try {
                      let docData = { name: value };
                      if (type === 'product-quick') {
                          const quickAddProductPrice = document.getElementById('quickAddProductPrice');
                          const quickAddProductUnit = document.getElementById('quickAddProductUnit');
                          docData.price = parseFloat(quickAddProductPrice ? quickAddProductPrice.value : '') || 0;
                          docData.unitId = quickAddProductUnit ? quickAddProductUnit.value : null;
                          docData.description = ''; // Default empty description
                          docData.currentStock = 0; // Default stock
                          docData.reorderPoint = 0; // Default reorder point
                          docData.supplierPrices = []; // Initialize supplier prices array for new product
                      } else if (type === 'supplier-quick') {
                          const quickAddSupplierContact = document.getElementById('quickAddSupplierContact');
                          docData.contact = quickAddSupplierContact ? quickAddSupplierContact.value.trim() : '';
                          docData.phone = '';
                          docData.email = '';
                          docData.address = '';
                      } else if (entityTypeField) {
                          docData.type = entityTypeField; // Set type for entities (sender/recipient)
                      }

                      const docRef = await addDoc(collection(window.db, `artifacts/${appId}/users/${window.userId}/${collectionName}`), docData);
                      window.showMessage('success', `${title.replace('mới', '')} đã được thêm thành công!`);

                      // If it was triggered from an autocomplete, update the input field
                      if (currentInputId) {
                          const targetSelect = document.getElementById(currentInputId);
                          if (targetSelect) {
                              // Add the new option and select it
                              const newOption = document.createElement('option');
                              newOption.value = docRef.id;
                              newOption.textContent = value;
                              targetSelect.appendChild(newOption);
                              targetSelect.value = docRef.id;
                              // Trigger change event to ensure other parts of the app react to the selection
                              targetSelect.dispatchEvent(new Event('change', { bubbles: true }));
                          }
                      }

                  } catch (e) {
                      console.error("Error adding document: ", e);
                      window.showMessage('error', `Lỗi khi thêm ${title.toLowerCase().replace('mới', '')}: ${e.message}`);
                  } finally {
                      window.hideLoadingOverlay();
                      window.closeQuickAddModal();
                  }
              };
          }
      };

      window.closeQuickAddModal = () => {
          const modal = document.getElementById('quickAddModal');
          if (modal) {
              modal.remove();
          }
      };

      // --- Helper Functions to get names by ID ---
      window.getUnitName = (unitId) => {
          const unit = window.units.find(u => u.id === unitId);
          return unit ? unit.name : 'N/A';
      };

      window.getCategoryName = (categoryId) => {
          const category = window.categories.find(c => c.id === categoryId);
          return category ? category.name : 'N/A';
      };

      window.getSupplierName = (supplierId) => {
          const supplier = window.suppliers.find(s => s.id === supplierId);
          return supplier ? supplier.name : 'N/A';
      };

      window.getProductName = (productId) => {
          const product = window.products.find(p => p.id === productId);
          return product ? product.name : 'N/A';
      };

      window.getBankName = (bankId) => {
          const bank = window.banks.find(b => b.id === bankId);
          return bank ? bank.name : 'N/A';
      };

      window.getEntityName = (entityId) => {
          const entity = window.entities.find(e => e.id === entityId);
          return entity ? entity.name : 'N/A';
      };

      // --- Populate Dropdowns (Select elements) ---
      window.populateUnitDropdowns = () => {
          document.querySelectorAll('select[data-type="unit"]').forEach(select => {
              const selectedValue = select.value; // Preserve current selection
              select.innerHTML = '<option value="">Chọn ĐVT</option>';
              window.units.forEach(unit => {
                  const option = document.createElement('option');
                  option.value = unit.id;
                  option.textContent = unit.name;
                  select.appendChild(option);
              });
              select.value = selectedValue; // Restore selection
          });
      };

      window.populateCategoryDropdowns = () => {
          document.querySelectorAll('select[data-type="category"]').forEach(select => {
              const selectedValue = select.value;
              select.innerHTML = '<option value="">Chọn Phân loại</option>';
              window.categories.forEach(category => {
                  const option = document.createElement('option');
                  option.value = category.id;
                  option.textContent = category.name;
                  select.appendChild(option);
              });
              select.value = selectedValue;
          });
      };

      window.populateSupplierDropdowns = () => {
          document.querySelectorAll('select[data-type="supplier"]').forEach(select => {
              const selectedValue = select.value;
              select.innerHTML = '<option value="">Chọn Nhà cung cấp</option>';
              window.suppliers.forEach(supplier => {
                  const option = document.createElement('option');
                  option.value = supplier.id;
                  option.textContent = supplier.name;
                  select.appendChild(option);
              });
              select.value = selectedValue;
          });
          // Also update dashboard supplier filter
          const dashboardSupplierSelect = document.getElementById('dashboard-supplier');
          if (dashboardSupplierSelect) {
              const selectedValue = dashboardSupplierSelect.value;
              dashboardSupplierSelect.innerHTML = '<option value="">Tất cả</option>';
              window.suppliers.forEach(supplier => {
                  const option = document.createElement('option');
                  option.value = supplier.id;
                  option.textContent = supplier.name;
                  dashboardSupplierSelect.appendChild(option);
              });
              dashboardSupplierSelect.value = selectedValue;
          }
          const debtFilterSupplierSelect = document.getElementById('debtFilterSupplier');
          if (debtFilterSupplierSelect) {
              const selectedValue = debtFilterSupplierSelect.value;
              debtFilterSupplierSelect.innerHTML = '<option value="">Tất cả</option>';
              window.suppliers.forEach(supplier => {
                  const option = document.createElement('option');
                  option.value = supplier.id;
                  option.textContent = supplier.name;
                  debtFilterSupplierSelect.appendChild(option);
              });
              debtFilterSupplierSelect.value = selectedValue;
          }
      };

      window.populateProductDropdowns = () => {
          document.querySelectorAll('select[data-type="product"]').forEach(select => {
              const selectedValue = select.value;
              select.innerHTML = '<option value="">Chọn Sản phẩm</option>';
              window.products.forEach(product => {
                  const option = document.createElement('option');
                  option.value = product.id;
                  option.textContent = product.name;
                  select.appendChild(option);
              });
              select.value = selectedValue;
          });
      };

      window.populateBankDropdowns = () => {
          document.querySelectorAll('select[data-type="bank"]').forEach(select => {
              const selectedValue = select.value;
              select.innerHTML = '<option value="">Chọn Ngân hàng</option>';
              window.banks.forEach(bank => {
                  const option = document.createElement('option');
                  option.value = bank.id;
                  option.textContent = bank.name;
                  select.appendChild(option);
              });
              select.value = selectedValue;
          });
      };

      window.populateEntityDropdowns = () => {
          document.querySelectorAll('select[data-type="entity-sender"]').forEach(select => {
              const selectedValue = select.value;
              select.innerHTML = '<option value="">Chọn Người gửi</option>';
              window.entities.filter(e => e.type === 'sender' || e.type === 'both').forEach(entity => {
                  const option = document.createElement('option');
                  option.value = entity.id;
                  option.textContent = entity.name;
                  select.appendChild(option);
              });
              select.value = selectedValue;
          });

          document.querySelectorAll('select[data-type="entity-recipient"]').forEach(select => {
              const selectedValue = select.value;
              select.innerHTML = '<option value="">Chọn Đơn vị nhận</option>';
              window.entities.filter(e => e.type === 'recipient' || e.type === 'both').forEach(entity => {
                  const option = document.createElement('option');
                  option.value = entity.id;
                  option.textContent = entity.name;
                  select.appendChild(option);
              });
              select.value = selectedValue;
          });
      };

      window.populatePaymentMethodDropdowns = () => {
          document.querySelectorAll('select[data-type="payment-method"]').forEach(select => {
              const selectedValue = select.value;
              select.innerHTML = `
                  <option value="">Chọn hình thức</option>
                  <option value="Tiền mặt">Tiền mặt</option>
                  <option value="SCB">SCB</option>
                  <option value="VCB-999">VCB-999</option>
                  <option value="VCB-7070">VCB-7070</option>
                  <option value="Thẻ tín dụng">Thẻ tín dụng</option>
              `;
              window.banks.forEach(bank => {
                  const option = document.createElement('option');
                  option.value = bank.name; // Use bank name as value for simplicity in payment method
                  option.textContent = bank.name;
                  select.appendChild(option);
              });
              select.value = selectedValue;
          });
      };


      // --- Tab: Product Management (Sản phẩm) ---
      let currentEditingProductId = null;

      window.renderProductList = () => {
          const productListBody = document.getElementById('productListBody');
          if (!productListBody) return;

          productListBody.innerHTML = '';
          if (window.products.length === 0) {
              productListBody.innerHTML = `<tr><td colspan="7" class="text-center text-gray-500 italic">Chưa có sản phẩm nào.</td></tr>`;
              return;
          }

          window.products.forEach(product => {
              const row = productListBody.insertRow();
              row.innerHTML = `
                  <td class="px-4 py-2">${product.name}</td>
                  <td class="px-4 py-2">${product.code || 'N/A'}</td>
                  <td class="px-4 py-2">${product.description || 'N/A'}</td>
                  <td class="px-4 py-2">${product.price ? product.price.toLocaleString('vi-VN') : '0'} VNĐ</td>
                  <td class="px-4 py-2">${window.getUnitName(product.unitId)}</td>
                  <td class="px-4 py-2">${window.getCategoryName(product.categoryId)}</td>
                  <td class="px-4 py-2 text-right">
                      <button class="bg-yellow-500 hover:bg-yellow-600 text-white px-3 py-1 rounded-md text-sm mr-2" onclick="window.editProduct('${product.id}')">Sửa</button>
                      <button class="bg-red-500 hover:bg-red-600 text-white px-3 py-1 rounded-md text-sm" onclick="window.deleteProduct('${product.id}')">Xóa</button>
                  </td>
              `;
          });
      };

      window.addOrUpdateProduct = async () => {
          const productName = document.getElementById('productName');
          const productCode = document.getElementById('productCode');
          const productDescription = document.getElementById('productDescription');
          const productPrice = document.getElementById('productPrice');
          const productUnit = document.getElementById('productUnit');
          const productCategory = document.getElementById('productCategory');
          const productReorderPoint = document.getElementById('productReorderPoint');

          const name = productName ? productName.value.trim() : '';
          const code = productCode ? productCode.value.trim() : '';
          const description = productDescription ? productDescription.value.trim() : '';
          const price = parseFloat(productPrice ? productPrice.value : '') || 0;
          const unitId = productUnit ? productUnit.value : '';
          const categoryId = productCategory ? productCategory.value : '';
          const reorderPoint = parseInt(productReorderPoint ? productReorderPoint.value : '') || 0;

          if (!name) {
              window.showMessage('error', 'Tên sản phẩm không được để trống.');
              return;
          }

          window.showLoadingOverlay();
          try {
              const productData = {
                  name, code, description, price, unitId, categoryId, reorderPoint,
                  currentStock: currentEditingProductId ? (window.products.find(p => p.id === currentEditingProductId)?.currentStock || 0) : 0, // Preserve stock on edit, set 0 on new
                  supplierPrices: currentEditingProductId ? (window.products.find(p => p.id === currentEditingProductId)?.supplierPrices || []) : [], // Preserve supplier prices on edit, initialize empty on new
                  updatedAt: new Date().toISOString()
              };

              if (currentEditingProductId) {
                  await updateDoc(doc(window.db, `artifacts/${appId}/users/${window.userId}/products`, currentEditingProductId), productData);
                  window.showMessage('success', 'Sản phẩm đã được cập nhật thành công!');
              } else {
                  productData.createdAt = new Date().toISOString();
                  await addDoc(collection(window.db, `artifacts/${appId}/users/${window.userId}/products`), productData);
                  window.showMessage('success', 'Sản phẩm đã được thêm thành công!');
              }
              window.clearProductForm();
          } catch (e) {
              console.error("Error adding/updating product: ", e);
              window.showMessage('error', `Lỗi khi lưu sản phẩm: ${e.message}`);
          } finally {
              window.hideLoadingOverlay();
          }
      };

      window.editProduct = (productId) => {
          const product = window.products.find(p => p.id === productId);
          if (product) {
              currentEditingProductId = productId;
              const productName = document.getElementById('productName');
              const productCode = document.getElementById('productCode');
              const productDescription = document.getElementById('productDescription');
              const productPrice = document.getElementById('productPrice');
              const productUnit = document.getElementById('productUnit');
              const productCategory = document.getElementById('productCategory');
              const productReorderPoint = document.getElementById('productReorderPoint');
              const productFormTitle = document.getElementById('productFormTitle');
              const addProductBtn = document.getElementById('addProductBtn');
              const cancelProductEditBtn = document.getElementById('cancelProductEditBtn');

              if (productName) productName.value = product.name;
              if (productCode) productCode.value = product.code || '';
              if (productDescription) productDescription.value = product.description || '';
              if (productPrice) productPrice.value = product.price || '';
              if (productUnit) productUnit.value = product.unitId || '';
              if (productCategory) productCategory.value = product.categoryId || '';
              if (productReorderPoint) productReorderPoint.value = product.reorderPoint || 0;
              if (productFormTitle) productFormTitle.textContent = 'Chỉnh Sửa Sản Phẩm';
              if (addProductBtn) addProductBtn.textContent = 'Cập Nhật Sản Phẩm';
              if (cancelProductEditBtn) cancelProductEditBtn.classList.remove('hidden');
          }
      };

      window.deleteProduct = (productId) => {
          window.showConfirmation('Bạn có chắc chắn muốn xóa sản phẩm này không?', async () => {
              window.showLoadingOverlay();
              try {
                  await deleteDoc(doc(window.db, `artifacts/${appId}/users/${window.userId}/products`, productId));
                  window.showMessage('success', 'Sản phẩm đã được xóa thành công!');
              } catch (e) {
                  console.error("Error deleting product: ", e);
                  window.showMessage('error', `Lỗi khi xóa sản phẩm: ${e.message}`);
              } finally {
                  window.hideLoadingOverlay();
              }
          });
      };

      window.clearProductForm = () => {
          currentEditingProductId = null;
          const productName = document.getElementById('productName');
          const productCode = document.getElementById('productCode');
          const productDescription = document.getElementById('productDescription');
          const productPrice = document.getElementById('productPrice');
          const productUnit = document.getElementById('productUnit');
          const productCategory = document.getElementById('productCategory');
          const productReorderPoint = document.getElementById('productReorderPoint');
          const productFormTitle = document.getElementById('productFormTitle');
          const addProductBtn = document.getElementById('addProductBtn');
          const cancelProductEditBtn = document.getElementById('cancelProductEditBtn');

          if (productName) productName.value = '';
          if (productCode) productCode.value = '';
          if (productDescription) productDescription.value = '';
          if (productPrice) productPrice.value = '';
          if (productUnit) productUnit.value = '';
          if (productCategory) productCategory.value = '';
          if (productReorderPoint) productReorderPoint.value = 0;
          if (productFormTitle) productFormTitle.textContent = 'Thêm Sản Phẩm Mới';
          if (addProductBtn) addProductBtn.textContent = 'Thêm Sản Phẩm';
          if (cancelProductEditBtn) cancelProductEditBtn.classList.add('hidden');
      };

      // --- Tab: Supplier Information (Thông tin NCC) ---
      let currentEditingSupplierId = null;

      window.renderSupplierList = () => {
          const supplierListBody = document.getElementById('supplierListBody');
          if (!supplierListBody) return;

          supplierListBody.innerHTML = '';
          if (window.suppliers.length === 0) {
              supplierListBody.innerHTML = `<tr><td colspan="7" class="text-center text-gray-500 italic">Chưa có nhà cung cấp nào.</td></tr>`;
              return;
          }

          window.suppliers.forEach(supplier => {
              const row = supplierListBody.insertRow();
              row.innerHTML = `
                  <td class="px-4 py-2">${supplier.name}</td>
                  <td class="px-4 py-2">${supplier.code || 'N/A'}</td>
                  <td class="px-4 py-2">${supplier.contact || 'N/A'}</td>
                  <td class="px-4 py-2">${supplier.phone || 'N/A'}</td>
                  <td class="px-4 py-2">${supplier.email || 'N/A'}</td>
                  <td class="px-4 py-2">${supplier.address || 'N/A'}</td>
                  <td class="px-4 py-2 text-right">
                      <button class="bg-yellow-500 hover:bg-yellow-600 text-white px-3 py-1 rounded-md text-sm mr-2" onclick="window.editSupplier('${supplier.id}')">Sửa</button>
                      <button class="bg-red-500 hover:bg-red-600 text-white px-3 py-1 rounded-md text-sm" onclick="window.deleteSupplier('${supplier.id}')">Xóa</button>
                  </td>
              `;
          });
      };

      window.addOrUpdateSupplier = async () => {
          const supplierName = document.getElementById('supplierName');
          const supplierCode = document.getElementById('supplierCode');
          const supplierContactPerson = document.getElementById('supplierContactPerson');
          const supplierPhone = document.getElementById('supplierPhone');
          const supplierEmail = document.getElementById('supplierEmail');
          const supplierAddress = document.getElementById('supplierAddress');

          const name = supplierName ? supplierName.value.trim() : '';
          const code = supplierCode ? supplierCode.value.trim() : '';
          const contact = supplierContactPerson ? supplierContactPerson.value.trim() : '';
          const phone = supplierPhone ? supplierPhone.value.trim() : '';
          const email = supplierEmail ? supplierEmail.value.trim() : '';
          const address = supplierAddress ? supplierAddress.value.trim() : '';

          if (!name) {
              window.showMessage('error', 'Tên nhà cung cấp không được để trống.');
              return;
          }

          window.showLoadingOverlay();
          try {
              const supplierData = {
                  name, code, contact, phone, email, address,
                  updatedAt: new Date().toISOString()
              };

              if (currentEditingSupplierId) {
                  await updateDoc(doc(window.db, `artifacts/${appId}/users/${window.userId}/suppliers`, currentEditingSupplierId), supplierData);
                  window.showMessage('success', 'Nhà cung cấp đã được cập nhật thành công!');
              } else {
                  supplierData.createdAt = new Date().toISOString();
                  await addDoc(collection(window.db, `artifacts/${appId}/users/${window.userId}/suppliers`), supplierData);
                  window.showMessage('success', 'Nhà cung cấp đã được thêm thành công!');
              }
              window.clearSupplierForm();
          } catch (e) {
              console.error("Error adding/updating supplier: ", e);
              window.showMessage('error', `Lỗi khi lưu nhà cung cấp: ${e.message}`);
          } finally {
              window.hideLoadingOverlay();
          }
      };

      window.editSupplier = (supplierId) => {
          const supplier = window.suppliers.find(s => s.id === supplierId);
          if (supplier) {
              currentEditingSupplierId = supplierId;
              const supplierName = document.getElementById('supplierName');
              const supplierCode = document.getElementById('supplierCode');
              const supplierContactPerson = document.getElementById('supplierContactPerson');
              const supplierPhone = document.getElementById('supplierPhone');
              const supplierEmail = document.getElementById('supplierEmail');
              const supplierAddress = document.getElementById('supplierAddress');
              const supplierFormTitle = document.getElementById('supplierFormTitle');
              const addSupplierBtn = document.getElementById('addSupplierBtn');
              const cancelSupplierEditBtn = document.getElementById('cancelSupplierEditBtn');

              if (supplierName) supplierName.value = supplier.name;
              if (supplierCode) supplierCode.value = supplier.code || '';
              if (supplierContactPerson) supplierContactPerson.value = supplier.contact || '';
              if (supplierPhone) supplierPhone.value = supplier.phone || '';
              if (supplierEmail) supplierEmail.value = supplier.email || '';
              if (supplierAddress) supplierAddress.value = supplier.address || '';
              if (supplierFormTitle) supplierFormTitle.textContent = 'Chỉnh Sửa Nhà Cung Cấp';
              if (addSupplierBtn) addSupplierBtn.textContent = 'Cập Nhật Nhà Cung Cấp';
              if (cancelSupplierEditBtn) cancelSupplierEditBtn.classList.remove('hidden');
          }
      };

      window.deleteSupplier = (supplierId) => {
          window.showConfirmation('Bạn có chắc chắn muốn xóa nhà cung cấp này không? Thao tác này sẽ không xóa các sản phẩm liên quan.', async () => {
              window.showLoadingOverlay();
              try {
                  await deleteDoc(doc(window.db, `artifacts/${appId}/users/${window.userId}/suppliers`, supplierId));
                  window.showMessage('success', 'Nhà cung cấp đã được xóa thành công!');
              } catch (e) {
                      console.error("Error deleting supplier: ", e);
                      window.showMessage('error', `Lỗi khi xóa nhà cung cấp: ${e.message}`);
                  } finally {
                      window.hideLoadingOverlay();
                  }
              });
          };

      window.clearSupplierForm = () => {
          currentEditingSupplierId = null;
          const supplierName = document.getElementById('supplierName');
          const supplierCode = document.getElementById('supplierCode');
          const supplierContactPerson = document.getElementById('supplierContactPerson');
          const supplierPhone = document.getElementById('supplierPhone');
          const supplierEmail = document.getElementById('supplierEmail');
          const supplierAddress = document.getElementById('supplierAddress');
          const supplierFormTitle = document.getElementById('supplierFormTitle');
          const addSupplierBtn = document.getElementById('addSupplierBtn');
          const cancelSupplierEditBtn = document.getElementById('cancelSupplierEditBtn');

          if (supplierName) supplierName.value = '';
          if (supplierCode) supplierCode.value = '';
          if (supplierContactPerson) supplierContactPerson.value = '';
          if (supplierPhone) supplierPhone.value = '';
          if (supplierEmail) supplierEmail.value = '';
          if (supplierAddress) supplierAddress.value = '';
          if (supplierFormTitle) supplierFormTitle.textContent = 'Thêm Nhà Cung Cấp Mới';
          if (addSupplierBtn) addSupplierBtn.textContent = 'Thêm Nhà Cung Cấp';
          if (cancelSupplierEditBtn) cancelSupplierEditBtn.classList.add('hidden');
      };

      // --- Tab: Order History (Lịch sử đặt hàng) ---
      let currentEditingOrderId = null;
      let currentOrderItems = []; // Stores items for the current order being added/edited

      window.renderOrderHistory = () => {
          const orderHistoryListBody = document.getElementById('orderHistoryListBody');
          if (!orderHistoryListBody) return;

          orderHistoryListBody.innerHTML = '';
          if (window.orders.length === 0) {
              orderHistoryListBody.innerHTML = `<tr><td colspan="9" class="text-center text-gray-500 italic">Chưa có lịch sử đặt hàng nào.</td></tr>`;
              return;
          }

          window.orders.forEach(order => {
              const row = orderHistoryListBody.insertRow();
              const remainingAmount = (order.totalAmount || 0) - (order.amountPaid || 0);
              row.innerHTML = `
                  <td class="px-4 py-2">${new Date(order.orderDate).toLocaleDateString('vi-VN')}</td>
                  <td class="px-4 py-2">${order.invoiceNumber || 'N/A'}</td>
                  <td class="px-4 py-2">${window.getSupplierName(order.supplierId)}</td>
                  <td class="px-4 py-2">
                      ${order.items.map(item => `${window.getProductName(item.productId)} (${item.quantity} ${window.getUnitName(window.products.find(p => p.id === item.productId)?.unitId || '')})`).join('<br>')}
                  </td>
                  <td class="px-4 py-2">${(order.totalAmount || 0).toLocaleString('vi-VN')} VNĐ</td>
                  <td class="px-4 py-2">${(order.amountPaid || 0).toLocaleString('vi-VN')} VNĐ</td>
                  <td class="px-4 py-2 ${remainingAmount > 0 ? 'text-red-600 font-semibold' : 'text-green-600'}">${remainingAmount.toLocaleString('vi-VN')} VNĐ</td>
                  <td class="px-4 py-2">${order.paymentStatus || 'N/A'}</td>
                  <td class="px-4 py-2 text-right">
                      <button class="bg-blue-500 hover:bg-blue-600 text-white px-3 py-1 rounded-md text-sm mr-2" onclick="window.openOrderDetailsModal('${order.id}')">Xem nhanh</button>
                      <button class="bg-yellow-500 hover:bg-yellow-600 text-white px-3 py-1 rounded-md text-sm mr-2" onclick="window.editOrder('${order.id}')">Sửa</button>
                      <button class="bg-red-500 hover:bg-red-600 text-white px-3 py-1 rounded-md text-sm" onclick="window.deleteOrder('${order.id}')">Xóa</button>
                  </td>
              `;
          });
      };

      window.addOrderItem = () => {
          // Add a new empty item, ensuring it has a unique temporary key for rendering if needed
          currentOrderItems.push({ productId: '', quantity: 0, unitPrice: 0, tempId: Date.now() + Math.random() });
          window.renderOrderItemsForm();
      };

      window.removeOrderItem = (index) => {
          currentOrderItems.splice(index, 1);
          window.renderOrderItemsForm();
      };

      window.updateOrderItem = (index, field, value) => {
          if (field === 'quantity' || field === 'unitPrice') {
              currentOrderItems[index][field] = parseFloat(value) || 0;
          } else {
              currentOrderItems[index][field] = value;
          }
          // Recalculate total amount for the form
          let totalAmount = 0;
          currentOrderItems.forEach(item => {
              totalAmount += (item.quantity || 0) * (item.unitPrice || 0);
          });
          const orderTotalAmount = document.getElementById('orderTotalAmount');
          if (orderTotalAmount) {
              orderTotalAmount.value = totalAmount.toLocaleString('vi-VN');
          }
      };

      window.renderOrderItemsForm = () => {
          const orderItemsContainer = document.getElementById('orderItemsContainer');
          if (!orderItemsContainer) return;

          orderItemsContainer.innerHTML = '';
          currentOrderItems.forEach((item, index) => {
              const itemRow = document.createElement('div');
              itemRow.className = 'inventory-item-row';
              itemRow.innerHTML = `
                  <div class="form-group flex-1">
                      <label for="orderProduct-${item.tempId || index}">Sản phẩm:</label>
                      <select id="orderProduct-${item.tempId || index}" data-type="product" class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white"></select>
                  </div>
                  <div class="form-group flex-1">
                      <label for="orderQuantity-${item.tempId || index}">Số lượng:</label>
                      <input type="number" id="orderQuantity-${item.tempId || index}" value="${item.quantity}" class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500">
                  </div>
                  <div class="form-group flex-1">
                      <label for="orderUnitPrice-${item.tempId || index}">Đơn giá:</label>
                      <input type="number" id="orderUnitPrice-${item.tempId || index}" value="${item.unitPrice}" class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500">
                  </div>
                  <button type="button" class="remove-item-btn" onclick="window.removeOrderItem(${index})">X</button>
              `;
              orderItemsContainer.appendChild(itemRow);

              // Populate product dropdown for this row
              const productSelect = document.getElementById(`orderProduct-${item.tempId || index}`);
              if (productSelect) {
                  productSelect.innerHTML = '<option value="">Chọn Sản phẩm</option>';
                  window.products.forEach(p => {
                      const option = document.createElement('option');
                      option.value = p.id;
                      option.textContent = p.name;
                      productSelect.appendChild(option);
                  });
                  productSelect.value = item.productId;

                  // Add event listeners for changes
                  productSelect.addEventListener('change', (e) => window.updateOrderItem(index, 'productId', e.target.value));
              }
              const orderQuantityInput = document.getElementById(`orderQuantity-${item.tempId || index}`);
              if (orderQuantityInput) {
                  orderQuantityInput.addEventListener('input', (e) => window.updateOrderItem(index, 'quantity', e.target.value));
              }
              const orderUnitPriceInput = document.getElementById(`orderUnitPrice-${item.tempId || index}`);
              if (orderUnitPriceInput) {
                  orderUnitPriceInput.addEventListener('input', (e) => window.updateOrderItem(index, 'unitPrice', e.target.value));
              }
          });
      };

      window.addOrUpdateOrder = async () => {
          const orderDateInput = document.getElementById('orderDate');
          const orderSupplierSelect = document.getElementById('orderSupplier');
          const orderAmountPaidInput = document.getElementById('orderAmountPaid');
          const orderPaymentStatusSelect = document.getElementById('orderPaymentStatus');
          const orderInvoiceNumberInput = document.getElementById('orderInvoiceNumber');
          const orderDeliveryDateInput = document.getElementById('orderDeliveryDate');
          const orderNotesInput = document.getElementById('orderNotes');

          const orderDate = orderDateInput ? orderDateInput.value : '';
          const supplierId = orderSupplierSelect ? orderSupplierSelect.value : '';
          const amountPaid = parseFloat(orderAmountPaidInput ? orderAmountPaidInput.value : '') || 0;
          const paymentStatus = orderPaymentStatusSelect ? orderPaymentStatusSelect.value : '';
          const invoiceNumber = orderInvoiceNumberInput ? orderInvoiceNumberInput.value.trim() : '';
          const deliveryDate = orderDeliveryDateInput ? orderDeliveryDateInput.value : '';
          const notes = orderNotesInput ? orderNotesInput.value.trim() : '';

          if (!orderDate || !supplierId || currentOrderItems.length === 0 || currentOrderItems.some(item => !item.productId || item.quantity <= 0 || item.unitPrice < 0)) {
              window.showMessage('error', 'Ngày đặt hàng, nhà cung cấp và ít nhất một sản phẩm với số lượng/đơn giá hợp lệ là bắt buộc.');
              return;
          }

          let totalAmount = 0;
          currentOrderItems.forEach(item => {
              totalAmount += (item.quantity || 0) * (item.unitPrice || 0);
          });

          window.showLoadingOverlay();
          try {
              const orderData = {
                  orderDate,
                  supplierId,
                  items: currentOrderItems.map(({tempId, ...rest}) => rest), // Remove tempId before saving
                  totalAmount,
                  amountPaid,
                  remainingAmount: totalAmount - amountPaid,
                  paymentStatus,
                  invoiceNumber,
                  deliveryDate,
                  notes,
                  updatedAt: new Date().toISOString()
              };

              if (currentEditingOrderId) {
                  await updateDoc(doc(window.db, `artifacts/${appId}/users/${window.userId}/orders`, currentEditingOrderId), orderData);
                  window.showMessage('success', 'Lịch sử đặt hàng đã được cập nhật thành công!');
              } else {
                  orderData.createdAt = new Date().toISOString();
                  const docRef = await addDoc(collection(window.db, `artifacts/${appId}/users/${window.userId}/orders`), orderData);
                  window.showMessage('success', 'Lịch sử đặt hàng đã được thêm thành công!');

                  // Logic: Order History -> Inventory
                  for (const item of currentOrderItems) {
                      const product = window.products.find(p => p.id === item.productId);
                      if (product) {
                          const newStock = (product.currentStock || 0) + (item.quantity || 0);
                          await updateDoc(doc(window.db, `artifacts/${appId}/users/${window.userId}/products`, item.productId), { currentStock: newStock });

                          // Add inventory movement record
                          await addDoc(collection(window.db, `artifacts/${appId}/users/${window.userId}/inventoryMovements`), {
                              date: new Date().toISOString(),
                              type: 'Nhập',
                              productId: item.productId,
                              quantity: item.quantity,
                              currentStockBefore: product.currentStock || 0,
                              currentStockAfter: newStock,
                              notes: `Nhập từ đơn hàng số ${docRef.id}`,
                              relatedOrderId: docRef.id,
                              createdAt: new Date().toISOString(),
                              items: [{ productId: item.productId, quantity: item.quantity }] // For multi-item structure
                          });
                      }
                  }

                  // Logic: Order History -> Công nợ (if not fully paid)
                  if (orderData.remainingAmount > 0) {
                      await addDoc(collection(window.db, `artifacts/${appId}/users/${window.userId}/debtRecords`), {
                          debtDate: orderDate,
                          supplierId: supplierId,
                          description: `Công nợ từ đơn hàng số ${docRef.id}`,
                          amount: orderData.remainingAmount,
                          dueDate: new Date(new Date(orderDate).setDate(new Date(orderDate).getDate() + 30)).toISOString().split('T')[0], // Default 30 days due, format YYYY-MM-DD
                          currentOutstanding: orderData.remainingAmount,
                          status: 'Chưa thanh toán',
                          paymentMethod: 'N/A', // Will be updated on payment
                          invoiceNumber: invoiceNumber,
                          relatedOrderId: docRef.id,
                          createdAt: new Date().toISOString()
                      });
                  }
              }
              window.clearOrderForm();
          } catch (e) {
              console.error("Error adding/updating order: ", e);
              window.showMessage('error', `Lỗi khi lưu lịch sử đặt hàng: ${e.message}`);
          } finally {
              window.hideLoadingOverlay();
          }
      };

      window.editOrder = (orderId) => {
          const order = window.orders.find(o => o.id === orderId);
          if (order) {
              currentEditingOrderId = orderId;
              const orderDateInput = document.getElementById('orderDate');
              const orderSupplierSelect = document.getElementById('orderSupplier');
              const orderAmountPaidInput = document.getElementById('orderAmountPaid');
              const orderPaymentStatusSelect = document.getElementById('orderPaymentStatus');
              const orderInvoiceNumberInput = document.getElementById('orderInvoiceNumber');
              const orderDeliveryDateInput = document.getElementById('orderDeliveryDate');
              const orderNotesInput = document.getElementById('orderNotes');
              const orderTotalAmountInput = document.getElementById('orderTotalAmount');
              const orderFormTitle = document.getElementById('orderFormTitle');
              const addOrderBtn = document.getElementById('addOrderBtn');
              const cancelOrderEditBtn = document.getElementById('cancelOrderEditBtn');

              if (orderDateInput) orderDateInput.value = order.orderDate;
              if (orderSupplierSelect) orderSupplierSelect.value = order.supplierId;
              if (orderAmountPaidInput) orderAmountPaidInput.value = order.amountPaid || 0;
              if (orderPaymentStatusSelect) orderPaymentStatusSelect.value = order.paymentStatus || '';
              if (orderInvoiceNumberInput) orderInvoiceNumberInput.value = order.invoiceNumber || '';
              if (orderDeliveryDateInput) orderDeliveryDateInput.value = order.deliveryDate || '';
              if (orderNotesInput) orderNotesInput.value = order.notes || '';
              if (orderTotalAmountInput) orderTotalAmountInput.value = (order.totalAmount || 0).toLocaleString('vi-VN');

              // Deep copy items and add tempId for rendering
              currentOrderItems = JSON.parse(JSON.stringify(order.items || [])).map(item => ({...item, tempId: Date.now() + Math.random()}));
              window.renderOrderItemsForm();

              if (orderFormTitle) orderFormTitle.textContent = 'Chỉnh Sửa Đơn Hàng';
              if (addOrderBtn) addOrderBtn.textContent = 'Cập Nhật Đơn Hàng';
              if (cancelOrderEditBtn) cancelOrderEditBtn.classList.remove('hidden');
          }
      };

      window.deleteOrder = (orderId) => {
          window.showConfirmation('Bạn có chắc chắn muốn xóa đơn hàng này không? Thao tác này sẽ hoàn tác số lượng tồn kho và công nợ liên quan.', async () => {
              window.showLoadingOverlay();
              try {
                  const orderToDelete = window.orders.find(o => o.id === orderId);
                  if (orderToDelete && orderToDelete.items) {
                      for (const item of orderToDelete.items) {
                          const product = window.products.find(p => p.id === item.productId);
                          if (product) {
                              let newStock = (product.currentStock || 0) - (item.quantity || 0);
                              await updateDoc(doc(window.db, `artifacts/${appId}/users/${window.userId}/products`, item.productId), { currentStock: Math.max(0, newStock) }); // Ensure stock doesn't go negative

                              // Delete related inventory movement (or mark as voided if preferred for history)
                              const q = query(collection(window.db, `artifacts/${appId}/users/${window.userId}/inventoryMovements`), where("relatedOrderId", "==", orderId));
                              const querySnapshot = await getDocs(q);
                              querySnapshot.forEach(async (docToDelete) => {
                                  await deleteDoc(doc(window.db, `artifacts/${appId}/users/${window.userId}/inventoryMovements`, docToDelete.id));
                              });
                          }
                      }
                  }

                  // Delete related debt records
                  const qDebt = query(collection(window.db, `artifacts/${appId}/users/${window.userId}/debtRecords`), where("relatedOrderId", "==", orderId));
                  const querySnapshotDebt = await getDocs(qDebt);
                  querySnapshotDebt.forEach(async (docToDelete) => {
                      await deleteDoc(doc(window.db, `artifacts/${appId}/users/${window.userId}/debtRecords`, docToDelete.id));
                  });

                  await deleteDoc(doc(window.db, `artifacts/${appId}/users/${window.userId}/orders`, orderId));
                  window.showMessage('success', 'Đơn hàng đã được xóa thành công!');
              } catch (e) {
                  console.error("Error deleting order: ", e);
                  window.showMessage('error', `Lỗi khi xóa đơn hàng: ${e.message}`);
              } finally {
                  window.hideLoadingOverlay();
              }
          });
      };

      window.clearOrderForm = () => {
          currentEditingOrderId = null;
          const orderDateInput = document.getElementById('orderDate');
          const orderSupplierSelect = document.getElementById('orderSupplier');
          const orderAmountPaidInput = document.getElementById('orderAmountPaid');
          const orderPaymentStatusSelect = document.getElementById('orderPaymentStatus');
          const orderInvoiceNumberInput = document.getElementById('orderInvoiceNumber');
          const orderDeliveryDateInput = document.getElementById('orderDeliveryDate');
          const orderNotesInput = document.getElementById('orderNotes');
          const orderTotalAmountInput = document.getElementById('orderTotalAmount');
          const orderFormTitle = document.getElementById('orderFormTitle');
          const addOrderBtn = document.getElementById('addOrderBtn');
          const cancelOrderEditBtn = document.getElementById('cancelOrderEditBtn');

          if (orderDateInput) orderDateInput.valueAsDate = new Date(); // Set to current date
          if (orderSupplierSelect) orderSupplierSelect.value = '';
          if (orderAmountPaidInput) orderAmountPaidInput.value = '';
          if (orderPaymentStatusSelect) orderPaymentStatusSelect.value = 'Chưa thanh toán';
          if (orderInvoiceNumberInput) orderInvoiceNumberInput.value = '';
          if (orderDeliveryDateInput) orderDeliveryDateInput.value = '';
          if (orderNotesInput) orderNotesInput.value = '';
          if (orderTotalAmountInput) orderTotalAmountInput.value = '0';
          currentOrderItems = [{ productId: '', quantity: 0, unitPrice: 0, tempId: Date.now() }]; // Reset with one empty item
          window.renderOrderItemsForm(); // Re-render form with empty item
          if (orderFormTitle) orderFormTitle.textContent = 'Thêm Đơn Hàng Mới';
          if (addOrderBtn) addOrderBtn.textContent = 'Thêm Đơn Hàng';
          if (cancelOrderEditBtn) cancelOrderEditBtn.classList.add('hidden');
      };

      // Initialize order form with one item row on load
      document.addEventListener('DOMContentLoaded', () => {
          window.clearOrderForm(); // Call this to set initial state for order form
      });

      // Modal for viewing order/inventory movement details
      window.openDetailsModal = (type, id) => {
          let record;
          let title = '';
          let contentHtml = '';

          if (type === 'order') {
              record = window.orders.find(o => o.id === id);
              title = 'Chi tiết Đơn hàng';
              if (record) {
                  contentHtml = `
                      <p><strong>Số hóa đơn:</strong> ${record.invoiceNumber || 'N/A'}</p>
                      <p><strong>Ngày đặt hàng:</strong> ${new Date(record.orderDate).toLocaleDateString('vi-VN')}</p>
                      <p><strong>Nhà cung cấp:</strong> ${window.getSupplierName(record.supplierId)}</p>
                      <p><strong>Tổng số tiền:</strong> ${(record.totalAmount || 0).toLocaleString('vi-VN')} VNĐ</p>
                      <p><strong>Số tiền đã thanh toán:</strong> ${(record.amountPaid || 0).toLocaleString('vi-VN')} VNĐ</p>
                      <p><strong>Số tiền còn lại:</strong> ${((record.totalAmount || 0) - (record.amountPaid || 0)).toLocaleString('vi-VN')} VNĐ</p>
                      <p><strong>Trạng thái thanh toán:</strong> ${record.paymentStatus || 'N/A'}</p>
                      <p><strong>Ngày giao hàng dự kiến:</strong> ${record.deliveryDate ? new Date(record.deliveryDate).toLocaleDateString('vi-VN') : 'N/A'}</p>
                      <p><strong>Ghi chú:</strong> ${record.notes || 'N/A'}</p>
                      <h4 class="font-semibold mt-4 mb-2 text-gray-700">Sản phẩm trong đơn hàng:</h4>
                      <table class="data-table w-full text-sm">
                          <thead>
                              <tr>
                                  <th>Sản phẩm</th>
                                  <th>Số lượng</th>
                                  <th>Đơn giá</th>
                                  <th>Thành tiền</th>
                              </tr>
                          </thead>
                          <tbody>
                              ${record.items.map(item => `
                                  <tr>
                                      <td>${window.getProductName(item.productId)}</td>
                                      <td>${item.quantity} ${window.getUnitName(window.products.find(p => p.id === item.productId)?.unitId || '')}</td>
                                      <td>${(item.unitPrice || 0).toLocaleString('vi-VN')} VNĐ</td>
                                      <td>${((item.quantity || 0) * (item.unitPrice || 0)).toLocaleString('vi-VN')} VNĐ</td>
                                  </tr>
                              `).join('')}
                          </tbody>
                      </table>
                  `;
              }
          } else if (type === 'inventoryMovement') {
              record = window.inventoryMovements.find(m => m.id === id);
              title = 'Chi tiết Giao dịch tồn kho';
              if (record) {
                  contentHtml = `
                      <p><strong>Ngày:</strong> ${new Date(record.date).toLocaleDateString('vi-VN')}</p>
                      <p><strong>Loại giao dịch:</strong> ${record.type}</p>
                      ${record.senderId ? `<p><strong>Người gửi:</strong> ${window.getEntityName(record.senderId)}</p>` : ''}
                      ${record.recipientId ? `<p><strong>Đơn vị nhận:</strong> ${window.getEntityName(record.recipientId)}</p>` : ''}
                      <p><strong>Ghi chú:</strong> ${record.notes || 'N/A'}</p>
                      <h4 class="font-semibold mt-4 mb-2 text-gray-700">Sản phẩm trong giao dịch:</h4>
                      <table class="data-table w-full text-sm">
                          <thead>
                              <tr>
                                  <th>Sản phẩm</th>
                                  <th>Số lượng</th>
                                  <th>Tồn trước</th>
                                  <th>Tồn sau</th>
                              </tr>
                          </thead>
                          <tbody>
                              ${record.items.map(item => {
                                  const product = window.products.find(p => p.id === item.productId);
                                  return `
                                      <tr>
                                          <td>${product ? product.name : 'N/A'}</td>
                                          <td>${item.quantity} ${product ? window.getUnitName(product.unitId) : ''}</td>
                                          <td>${item.currentStockBefore !== undefined ? item.currentStockBefore : 'N/A'}</td>
                                          <td>${item.currentStockAfter !== undefined ? item.currentStockAfter : 'N/A'}</td>
                                      </tr>
                                  `;
                              }).join('')}
                          </tbody>
                      </table>
                  `;
              }
          }

          if (!record) {
              window.showMessage('error', 'Không tìm thấy chi tiết cho bản ghi này.');
              return;
          }

          const modalHtml = `
              <div id="detailsModal" class="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-[7000]">
                  <div class="bg-white p-6 rounded-lg shadow-xl w-full max-w-2xl max-h-[90vh] overflow-y-auto">
                      <h3 class="text-xl font-semibold mb-4 text-gray-700">${title}</h3>
                      <div class="text-gray-600 text-sm">
                          ${contentHtml}
                      </div>
                      <div class="flex justify-end gap-4 mt-6">
                          <button type="button" class="btn btn-secondary" onclick="window.closeDetailsModal()">Đóng</button>
                      </div>
                  </div>
              </div>
          `;
          document.body.insertAdjacentHTML('beforeend', modalHtml);
      };

      window.closeDetailsModal = () => {
          const modal = document.getElementById('detailsModal');
          if (modal) {
              modal.remove();
          }
      };


      // --- Tab: Inventory (Tồn kho) ---
      let currentInventoryMovementItems = []; // Stores items for the current inventory movement being added/edited

      window.renderInventoryList = () => {
          const inventoryListBody = document.getElementById('inventoryListBody');
          if (!inventoryListBody) return;

          inventoryListBody.innerHTML = '';
          if (window.inventoryMovements.length === 0) {
              inventoryListBody.innerHTML = `<tr><td colspan="8" class="text-center text-gray-500 italic">Chưa có lịch sử xuất/nhập/bàn giao nào.</td></tr>`;
              return;
          }

          window.inventoryMovements.forEach(movement => {
              const row = inventoryListBody.insertRow();
              const productDetails = movement.items.map(item => {
                  const product = window.products.find(p => p.id === item.productId);
                  return `${product ? product.name : 'N/A'} (${item.quantity} ${product ? window.getUnitName(product.unitId) : ''})`;
              }).join('<br>');

              row.innerHTML = `
                  <td class="px-4 py-2">${new Date(movement.date).toLocaleDateString('vi-VN')}</td>
                  <td class="px-4 py-2">${movement.type}</td>
                  <td class="px-4 py-2">${productDetails}</td>
                  <td class="px-4 py-2">${movement.senderId ? window.getEntityName(movement.senderId) : 'N/A'}</td>
                  <td class="px-4 py-2">${movement.recipientId ? window.getEntityName(movement.recipientId) : 'N/A'}</td>
                  <td class="px-4 py-2">${movement.notes || 'N/A'}</td>
                  <td class="px-4 py-2 text-right">
                      <button class="bg-blue-500 hover:bg-blue-600 text-white px-3 py-1 rounded-md text-sm mr-2" onclick="window.openDetailsModal('inventoryMovement', '${movement.id}')">Xem nhanh</button>
                      <button class="bg-red-500 hover:bg-red-600 text-white px-3 py-1 rounded-md text-sm" onclick="window.deleteInventoryMovement('${movement.id}')">Xóa</button>
                  </td>
              `;
          });

          // Update product stock overview
          const productStockOverviewBody = document.getElementById('productStockOverviewBody');
          if (productStockOverviewBody) {
              productStockOverviewBody.innerHTML = '';
              if (window.products.length === 0) {
                  productStockOverviewBody.innerHTML = `<tr><td colspan="5" class="text-center text-gray-500 italic">Chưa có sản phẩm nào trong kho.</td></tr>`;
                  return;
              }
              window.products.forEach(product => {
                  const isLowStock = product.reorderPoint > 0 && product.currentStock <= product.reorderPoint;
                  const row = productStockOverviewBody.insertRow();
                  row.className = isLowStock ? 'bg-red-100' : ''; // Highlight low stock
                  row.innerHTML = `
                      <td class="px-4 py-2">${product.name}</td>
                      <td class="px-4 py-2">${window.getUnitName(product.unitId)}</td>
                      <td class="px-4 py-2">${(product.currentStock || 0).toLocaleString('vi-VN')}</td>
                      <td class="px-4 py-2">${(product.reorderPoint || 0).toLocaleString('vi-VN')}</td>
                      <td class="px-4 py-2 ${isLowStock ? 'text-red-600 font-bold' : ''}">${isLowStock ? 'Cần nhập thêm!' : 'Ổn định'}</td>
                  `;
              });
          }
      };

      window.addInventoryItem = () => {
          currentInventoryMovementItems.push({ productId: '', quantity: 0, tempId: Date.now() + Math.random() });
          window.renderInventoryMovementItemsForm();
      };

      window.removeInventoryItem = (index) => {
          currentInventoryMovementItems.splice(index, 1);
          window.renderInventoryMovementItemsForm();
      };

      window.updateInventoryItem = (index, field, value) => {
          if (field === 'quantity') {
              currentInventoryMovementItems[index][field] = parseFloat(value) || 0;
          } else {
              currentInventoryMovementItems[index][field] = value;
          }
      };

      window.renderInventoryMovementItemsForm = () => {
          const inventoryItemsContainer = document.getElementById('inventoryItemsContainer');
          if (!inventoryItemsContainer) return;

          inventoryItemsContainer.innerHTML = '';
          currentInventoryMovementItems.forEach((item, index) => {
              const itemRow = document.createElement('div');
              itemRow.className = 'inventory-item-row';
              itemRow.innerHTML = `
                  <div class="form-group flex-1">
                      <label for="invProduct-${item.tempId || index}">Sản phẩm:</label>
                      <select id="invProduct-${item.tempId || index}" data-type="product" class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white"></select>
                  </div>
                  <div class="form-group flex-1">
                      <label for="invQuantity-${item.tempId || index}">Số lượng:</label>
                      <input type="number" id="invQuantity-${item.tempId || index}" value="${item.quantity}" class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500">
                  </div>
                  <button type="button" class="remove-item-btn" onclick="window.removeInventoryItem(${index})">X</button>
              `;
              inventoryItemsContainer.appendChild(itemRow);

              const productSelect = document.getElementById(`invProduct-${item.tempId || index}`);
              if (productSelect) {
                  productSelect.innerHTML = '<option value="">Chọn Sản phẩm</option>';
                  window.products.forEach(p => {
                      const option = document.createElement('option');
                      option.value = p.id;
                      option.textContent = p.name;
                      productSelect.appendChild(option);
                  });
                  productSelect.value = item.productId;
                  productSelect.addEventListener('change', (e) => window.updateInventoryItem(index, 'productId', e.target.value));
              }
              const quantityInput = document.getElementById(`invQuantity-${item.tempId || index}`);
              if (quantityInput) {
                  quantityInput.addEventListener('input', (e) => window.updateInventoryItem(index, 'quantity', e.target.value));
              }
          });
      };

      window.addInventoryMovement = async () => {
          const inventoryMovementDateInput = document.getElementById('inventoryMovementDate');
          const inventoryMovementTypeSelect = document.getElementById('inventoryMovementType');
          const inventoryMovementSenderSelect = document.getElementById('inventoryMovementSender');
          const inventoryMovementRecipientSelect = document.getElementById('inventoryMovementRecipient');
          const inventoryMovementNotesInput = document.getElementById('inventoryMovementNotes');

          const date = inventoryMovementDateInput ? inventoryMovementDateInput.value : '';
          const type = inventoryMovementTypeSelect ? inventoryMovementTypeSelect.value : '';
          const senderId = inventoryMovementSenderSelect ? inventoryMovementSenderSelect.value : null;
          const recipientId = inventoryMovementRecipientSelect ? inventoryMovementRecipientSelect.value : null;
          const notes = inventoryMovementNotesInput ? inventoryMovementNotesInput.value.trim() : '';

          if (!date || !type || currentInventoryMovementItems.length === 0 || currentInventoryMovementItems.some(item => !item.productId || item.quantity <= 0)) {
              window.showMessage('error', 'Vui lòng điền đầy đủ thông tin: Ngày, Loại, và ít nhất một sản phẩm với số lượng hợp lệ.');
              return;
          }

          if ((type === 'Xuất' || type === 'Bàn giao') && !senderId) {
              window.showMessage('error', 'Vui lòng chọn Người gửi cho giao dịch Xuất/Bàn giao.');
              return;
          }

          if (type === 'Bàn giao' && !recipientId) {
              window.showMessage('error', 'Vui lòng chọn Đơn vị nhận hàng cho giao dịch Bàn giao.');
              return;
          }

          window.showLoadingOverlay();
          try {
              const movementData = {
                  date, type, notes,
                  senderId: type !== 'Nhập' ? senderId : null, // Sender only for Xuất/Bàn giao
                  recipientId: type === 'Bàn giao' ? recipientId : null, // Recipient only for Bàn giao
                  items: [], // Store items with their stock before/after
                  createdAt: new Date().toISOString()
              };

              for (const item of currentInventoryMovementItems) {
                  const product = window.products.find(p => p.id === item.productId);
                  if (!product) {
                      window.showMessage('error', `Sản phẩm "${item.productId}" không tồn tại.`);
                      window.hideLoadingOverlay();
                      return;
                  }

                  let newStock = product.currentStock;
                  if (type === 'Nhập') {
                      newStock += item.quantity;
                  } else if (type === 'Xuất' || type === 'Bàn giao') {
                      if (newStock < item.quantity) {
                          window.showMessage('error', `Số lượng xuất/bàn giao cho sản phẩm "${product.name}" vượt quá số lượng tồn kho.`);
                          window.hideLoadingOverlay();
                          return;
                      }
                      newStock -= item.quantity;
                  }

                  movementData.items.push({
                      productId: item.productId,
                      quantity: item.quantity,
                      currentStockBefore: product.currentStock,
                      currentStockAfter: newStock
                  });

                  // Update product stock in Firestore
                  await updateDoc(doc(window.db, `artifacts/${appId}/users/${window.userId}/products`, item.productId), { currentStock: newStock });
              }

              await addDoc(collection(window.db, `artifacts/${appId}/users/${window.userId}/inventoryMovements`), movementData);
              window.showMessage('success', `Đã ghi nhận giao dịch tồn kho thành công.`);
              window.clearInventoryMovementForm();
          } catch (e) {
              console.error("Error adding inventory movement: ", e);
              window.showMessage('error', `Lỗi khi ghi nhận thao tác tồn kho: ${e.message}`);
          } finally {
              window.hideLoadingOverlay();
          }
      };

      window.deleteInventoryMovement = (movementId) => {
          window.showConfirmation('Bạn có chắc chắn muốn xóa lịch sử xuất/nhập/bàn giao này không? Thao tác này sẽ hoàn tác số lượng tồn kho.', async () => {
              window.showLoadingOverlay();
              try {
                  const movementToDelete = window.inventoryMovements.find(m => m.id === movementId);
                  if (movementToDelete && movementToDelete.items) {
                      for (const item of movementToDelete.items) {
                          const product = window.products.find(p => p.id === item.productId);
                          if (product) {
                              let revertedStock = product.currentStock;
                              if (movementToDelete.type === 'Nhập') {
                                  revertedStock -= item.quantity;
                              } else if (movementToDelete.type === 'Xuất' || movementToDelete.type === 'Bàn giao') {
                                  revertedStock += item.quantity;
                              }
                              await updateDoc(doc(window.db, `artifacts/${appId}/users/${window.userId}/products`, item.productId), { currentStock: Math.max(0, revertedStock) });
                          }
                      }
                  }

                  await deleteDoc(doc(window.db, `artifacts/${appId}/users/${window.userId}/inventoryMovements`, movementId));
                  window.showMessage('success', 'Lịch sử xuất/nhập/bàn giao đã được xóa thành công!');
              } catch (e) {
                  console.error("Error deleting inventory movement: ", e);
                  window.showMessage('error', `Lỗi khi xóa lịch sử tồn kho: ${e.message}`);
              } finally {
                  window.hideLoadingOverlay();
              }
          });
      };

      window.clearInventoryMovementForm = () => {
          const inventoryMovementDateInput = document.getElementById('inventoryMovementDate');
          const inventoryMovementTypeSelect = document.getElementById('inventoryMovementType');
          const inventoryMovementSenderSelect = document.getElementById('inventoryMovementSender');
          const inventoryMovementRecipientSelect = document.getElementById('inventoryMovementRecipient');
          const inventoryMovementNotesInput = document.getElementById('inventoryMovementNotes');

          if (inventoryMovementDateInput) inventoryMovementDateInput.valueAsDate = new Date();
          if (inventoryMovementTypeSelect) {
              inventoryMovementTypeSelect.value = 'Nhập';
              window.toggleInventoryMovementFields(); // Ensure correct fields are visible
          }
          if (inventoryMovementSenderSelect) inventoryMovementSenderSelect.value = '';
          if (inventoryMovementRecipientSelect) inventoryMovementRecipientSelect.value = '';
          if (inventoryMovementNotesInput) inventoryMovementNotesInput.value = '';

          currentInventoryMovementItems = [{ productId: '', quantity: 0, tempId: Date.now() }]; // Reset with one empty item
          window.renderInventoryMovementItemsForm();
      };

      // Toggle visibility of sender/recipient fields based on movement type
      window.toggleInventoryMovementFields = () => {
          const type = document.getElementById('inventoryMovementType')?.value;
          const senderGroup = document.getElementById('inventorySenderGroup');
          const recipientGroup = document.getElementById('inventoryRecipientGroup');

          if (senderGroup) {
              senderGroup.classList.toggle('hidden', type === 'Nhập');
          }
          if (recipientGroup) {
              recipientGroup.classList.toggle('hidden', type !== 'Bàn giao');
          }
      };

      // Initial setup for inventory form
      document.addEventListener('DOMContentLoaded', () => {
          window.clearInventoryMovementForm(); // Initialize form
          const inventoryMovementTypeSelect = document.getElementById('inventoryMovementType');
          if (inventoryMovementTypeSelect) {
              inventoryMovementTypeSelect.addEventListener('change', window.toggleInventoryMovementFields);
          }
          window.toggleInventoryMovementFields(); // Set initial visibility
      });


      // --- Tab: Debt Management (Công nợ) ---
      let currentEditingDebtId = null;

      window.renderDebtList = () => {
          const debtListBody = document.getElementById('debtListBody');
          if (!debtListBody) return;

          debtListBody.innerHTML = '';
          if (window.debtRecords.length === 0) {
              debtListBody.innerHTML = `<tr><td colspan="8" class="text-center text-gray-500 italic">Chưa có công nợ nào.</td></tr>`;
              return;
          }

          window.debtRecords.forEach(debt => {
              const row = debtListBody.insertRow();
              let statusClass = '';
              // Update status based on current date
              let calculatedStatus = debt.status;
              if (debt.currentOutstanding > 0) {
                  const today = new Date();
                  today.setHours(0, 0, 0, 0); // Normalize to start of day
                  const dueDate = new Date(debt.dueDate);
                  dueDate.setHours(0, 0, 0, 0); // Normalize to start of day

                  if (debt.currentOutstanding <= 0) { // Should be newOutstanding, but using currentOutstanding for display
                      calculatedStatus = 'Đã thanh toán';
                  } else if (debt.amount > debt.currentOutstanding && debt.currentOutstanding > 0) {
                      calculatedStatus = 'Đã thanh toán một phần';
                  } else if (dueDate < today) {
                      calculatedStatus = 'Quá hạn';
                  } else {
                      calculatedStatus = 'Chưa thanh toán';
                  }
              } else {
                  calculatedStatus = 'Đã thanh toán';
              }

              if (calculatedStatus === 'Quá hạn') {
                  statusClass = 'status-overdue';
              } else if (calculatedStatus === 'Chưa thanh toán' && new Date(debt.dueDate) < new Date(new Date().setDate(new Date().getDate() + 7))) {
                  statusClass = 'status-due-soon'; // Due within 7 days
              } else if (calculatedStatus === 'Đã thanh toán') {
                  statusClass = 'status-on-time';
              } else if (calculatedStatus === 'Đã thanh toán một phần') {
                  statusClass = 'text-blue-600 font-semibold'; // Custom class for partial payment
              }

              row.innerHTML = `
                  <td class="px-4 py-2">${new Date(debt.debtDate).toLocaleDateString('vi-VN')}</td>
                  <td class="px-4 py-2">${window.getSupplierName(debt.supplierId)}</td>
                  <td class="px-4 py-2">${debt.description || 'N/A'}</td>
                  <td class="px-4 py-2">${(debt.amount || 0).toLocaleString('vi-VN')} VNĐ</td>
                  <td class="px-4 py-2">${(debt.currentOutstanding || 0).toLocaleString('vi-VN')} VNĐ</td>
                  <td class="px-4 py-2">${new Date(debt.dueDate).toLocaleDateString('vi-VN')}</td>
                  <td class="px-4 py-2 ${statusClass}">${calculatedStatus}</td>
                  <td class="px-4 py-2 text-right">
                      <button class="bg-blue-500 hover:bg-blue-600 text-white px-3 py-1 rounded-md text-sm mr-2" onclick="window.openPayDebtModal('${debt.id}')">Thanh toán</button>
                      <button class="bg-yellow-500 hover:bg-yellow-600 text-white px-3 py-1 rounded-md text-sm mr-2" onclick="window.editDebt('${debt.id}')">Sửa</button>
                      <button class="bg-red-500 hover:bg-red-600 text-white px-3 py-1 rounded-md text-sm" onclick="window.deleteDebt('${debt.id}')">Xóa</button>
                  </td>
              `;
          });
      };

      window.addOrUpdateDebt = async () => {
          const debtDateInput = document.getElementById('debtDate');
          const debtSupplierSelect = document.getElementById('debtSupplier');
          const debtDescriptionInput = document.getElementById('debtDescription');
          const debtAmountInput = document.getElementById('debtAmount');
          const debtDueDateInput = document.getElementById('debtDueDate');
          const debtPaymentMethodSelect = document.getElementById('debtPaymentMethod');
          const debtInvoiceNumberInput = document.getElementById('debtInvoiceNumber');

          const debtDate = debtDateInput ? debtDateInput.value : '';
          const supplierId = debtSupplierSelect ? debtSupplierSelect.value : '';
          const description = debtDescriptionInput ? debtDescriptionInput.value.trim() : '';
          const amount = parseFloat(debtAmountInput ? debtAmountInput.value : '') || 0;
          const dueDate = debtDueDateInput ? debtDueDateInput.value : '';
          const paymentMethod = debtPaymentMethodSelect ? debtPaymentMethodSelect.value : '';
          const invoiceNumber = debtInvoiceNumberInput ? debtInvoiceNumberInput.value.trim() : '';

          if (!debtDate || !supplierId || !amount || !dueDate) {
              window.showMessage('error', 'Vui lòng điền đầy đủ thông tin công nợ bắt buộc.');
              return;
          }

          window.showLoadingOverlay();
          try {
              let debtData = {
                  debtDate, supplierId, description, amount, dueDate, paymentMethod, invoiceNumber,
                  updatedAt: new Date().toISOString()
              };

              if (currentEditingDebtId) {
                  const existingDebt = window.debtRecords.find(d => d.id === currentEditingDebtId);
                  if (existingDebt) {
                      debtData.currentOutstanding = existingDebt.currentOutstanding; // Preserve outstanding
                      debtData.payments = existingDebt.payments; // Preserve payments
                      // Recalculate status based on new amount and current outstanding
                      if (debtData.currentOutstanding <= 0) {
                          debtData.status = 'Đã thanh toán';
                      } else if (debtData.amount > debtData.currentOutstanding && debtData.currentOutstanding > 0) {
                          debtData.status = 'Đã thanh toán một phần';
                      }
                      else if (new Date(debtData.dueDate) < new Date() && debtData.currentOutstanding > 0) {
                          debtData.status = 'Quá hạn';
                      }
                      else {
                          debtData.status = 'Chưa thanh toán';
                      }
                  }
                  await updateDoc(doc(window.db, `artifacts/${appId}/users/${window.userId}/debtRecords`, currentEditingDebtId), debtData);
                  window.showMessage('success', 'Công nợ đã được cập nhật thành công!');
              } else {
                  debtData.currentOutstanding = amount; // For new debt, outstanding is full amount
                  debtData.payments = []; // Initialize payments array
                  debtData.status = 'Chưa thanh toán';
                  debtData.createdAt = new Date().toISOString();
                  await addDoc(collection(window.db, `artifacts/${appId}/users/${window.userId}/debtRecords`), debtData);
                  window.showMessage('success', 'Công nợ đã được thêm thành công!');
              }
              window.clearDebtForm();
          } catch (e) {
              console.error("Error adding/updating debt: ", e);
              window.showMessage('error', `Lỗi khi lưu công nợ: ${e.message}`);
          } finally {
              window.hideLoadingOverlay();
          }
      };

      window.editDebt = (debtId) => {
          const debt = window.debtRecords.find(d => d.id === debtId);
          if (debt) {
              currentEditingDebtId = debtId;
              const debtDateInput = document.getElementById('debtDate');
              const debtSupplierSelect = document.getElementById('debtSupplier');
              const debtDescriptionInput = document.getElementById('debtDescription');
              const debtAmountInput = document.getElementById('debtAmount');
              const debtDueDateInput = document.getElementById('debtDueDate');
              const debtPaymentMethodSelect = document.getElementById('debtPaymentMethod');
              const debtInvoiceNumberInput = document.getElementById('debtInvoiceNumber');
              const debtFormTitle = document.getElementById('debtFormTitle');
              const addDebtBtn = document.getElementById('addDebtBtn');
              const cancelDebtEditBtn = document.getElementById('cancelDebtEditBtn');

              if (debtDateInput) debtDateInput.value = debt.debtDate;
              if (debtSupplierSelect) debtSupplierSelect.value = debt.supplierId;
              if (debtDescriptionInput) debtDescriptionInput.value = debt.description || '';
              if (debtAmountInput) debtAmountInput.value = debt.amount || '';
              if (debtDueDateInput) debtDueDateInput.value = debt.dueDate;
              if (debtPaymentMethodSelect) debtPaymentMethodSelect.value = debt.paymentMethod || '';
              if (debtInvoiceNumberInput) debtInvoiceNumberInput.value = debt.invoiceNumber || '';

              if (debtFormTitle) debtFormTitle.textContent = 'Chỉnh Sửa Công Nợ';
              if (addDebtBtn) addDebtBtn.textContent = 'Cập Nhật Công Nợ';
              if (cancelDebtEditBtn) cancelDebtEditBtn.classList.remove('hidden');
          }
      };

      window.deleteDebt = (debtId) => {
          window.showConfirmation('Bạn có chắc chắn muốn xóa công nợ này không?', async () => {
              window.showLoadingOverlay();
              try {
                  await deleteDoc(doc(window.db, `artifacts/${appId}/users/${window.userId}/debtRecords`, debtId));
                  window.showMessage('success', 'Công nợ đã được xóa thành công!');
              } catch (e) {
                  console.error("Error deleting debt: ", e);
                  window.showMessage('error', `Lỗi khi xóa công nợ: ${e.message}`);
              } finally {
                  window.hideLoadingOverlay();
              }
          });
      };

      window.clearDebtForm = () => {
          currentEditingDebtId = null;
          const debtDateInput = document.getElementById('debtDate');
          const debtSupplierSelect = document.getElementById('debtSupplier');
          const debtDescriptionInput = document.getElementById('debtDescription');
          const debtAmountInput = document.getElementById('debtAmount');
          const debtDueDateInput = document.getElementById('debtDueDate');
          const debtPaymentMethodSelect = document.getElementById('debtPaymentMethod');
          const debtInvoiceNumberInput = document.getElementById('debtInvoiceNumber');
          const debtFormTitle = document.getElementById('debtFormTitle');
          const addDebtBtn = document.getElementById('addDebtBtn');
          const cancelDebtEditBtn = document.getElementById('cancelDebtEditBtn');

          if (debtDateInput) debtDateInput.valueAsDate = new Date();
          if (debtSupplierSelect) debtSupplierSelect.value = '';
          if (debtDescriptionInput) debtDescriptionInput.value = '';
          if (debtAmountInput) debtAmountInput.value = '';
          if (debtDueDateInput) debtDueDateInput.value = '';
          if (debtPaymentMethodSelect) debtPaymentMethodSelect.value = '';
          if (debtInvoiceNumberInput) debtInvoiceNumberInput.value = '';
          if (debtFormTitle) debtFormTitle.textContent = 'Thêm Công Nợ Mới';
          if (addDebtBtn) addDebtBtn.textContent = 'Thêm Công Nợ';
          if (cancelDebtEditBtn) cancelDebtEditBtn.classList.add('hidden');
      };

      // Debt Payment Modal
      window.openPayDebtModal = (debtId) => {
          const debt = window.debtRecords.find(d => d.id === debtId);
          if (!debt) {
              window.showMessage('error', 'Không tìm thấy công nợ.');
              return;
          }

          const modalHtml = `
              <div id="payDebtModal" class="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-[7000]">
                  <div class="bg-white p-6 rounded-lg shadow-xl w-full max-w-md">
                      <h3 class="text-xl font-semibold mb-4 text-gray-700">Thanh toán công nợ</h3>
                      <p class="mb-2 text-gray-700">NCC: <span class="font-medium">${window.getSupplierName(debt.supplierId)}</span></p>
                      <p class="mb-2 text-gray-700">Mô tả: <span class="font-medium">${debt.description || 'N/A'}</span></p>
                      <p class="mb-2 text-gray-700">Số tiền nợ gốc: <span class="font-medium">${(debt.amount || 0).toLocaleString('vi-VN')} VNĐ</span></p>
                      <p class="mb-4 text-gray-700">Số tiền còn lại: <span class="font-bold text-red-600">${(debt.currentOutstanding || 0).toLocaleString('vi-VN')} VNĐ</span></p>

                      <div class="form-group mb-4">
                          <label for="paymentAmount">Số tiền thanh toán:</label>
                          <input type="number" id="paymentAmount" placeholder="Nhập số tiền thanh toán" class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500" required>
                      </div>
                      <div class="form-group mb-4">
                          <label for="paymentDate">Ngày thanh toán:</label>
                          <input type="date" id="paymentDate" class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500" value="${new Date().toISOString().split('T')[0]}" required>
                      </div>
                      <div class="form-group mb-4">
                          <label for="paymentMethod">Hình thức thanh toán:</label>
                          <select id="paymentMethod" data-type="payment-method" class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white">
                              <!-- Options populated by JS -->
                          </select>
                      </div>
                      <div class="form-group mb-4">
                          <label for="paymentNotes">Ghi chú:</label>
                          <textarea id="paymentNotes" rows="2" class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 resize-y"></textarea>
                      </div>
                      <div class="flex justify-end gap-4">
                          <button type="button" class="btn btn-secondary" onclick="window.closePayDebtModal()">Hủy</button>
                          <button type="button" class="btn btn-primary" id="confirmPayDebtBtn">Thanh toán</button>
                      </div>
                  </div>
              </div>
          `;
          document.body.insertAdjacentHTML('beforeend', modalHtml);
          window.populatePaymentMethodDropdowns(); // Populate dropdown after modal is in DOM

          const confirmPayDebtBtn = document.getElementById('confirmPayDebtBtn');
          if (confirmPayDebtBtn) {
              confirmPayDebtBtn.onclick = async () => {
                  const paymentAmountInput = document.getElementById('paymentAmount');
                  const paymentDateInput = document.getElementById('paymentDate');
                  const paymentMethodSelect = document.getElementById('paymentMethod');
                  const paymentNotesInput = document.getElementById('paymentNotes');

                  const paymentAmount = parseFloat(paymentAmountInput ? paymentAmountInput.value : '');
                  const paymentDate = paymentDateInput ? paymentDateInput.value : '';
                  const paymentMethod = paymentMethodSelect ? paymentMethodSelect.value : '';
                  const paymentNotes = paymentNotesInput ? paymentNotesInput.value.trim() : '';

                  if (isNaN(paymentAmount) || paymentAmount <= 0) {
                      window.showMessage('error', 'Số tiền thanh toán phải là một số dương.');
                      return;
                  }
                  if (paymentAmount > debt.currentOutstanding) {
                      window.showMessage('error', 'Số tiền thanh toán không được vượt quá số tiền còn lại.');
                      return;
                  }

                  window.showLoadingOverlay();
                  try {
                      const newOutstanding = debt.currentOutstanding - paymentAmount;
                      const updatedPayments = [...(debt.payments || []), {
                          paymentDate,
                          amount: paymentAmount,
                          method: paymentMethod,
                          notes: paymentNotes,
                          recordedAt: new Date().toISOString()
                      }];

                      let newStatus = 'Chưa thanh toán';
                      if (newOutstanding <= 0) {
                          newStatus = 'Đã thanh toán';
                      } else if (debt.amount > newOutstanding && newOutstanding > 0) {
                          newStatus = 'Đã thanh toán một phần';
                      } else if (new Date(debt.dueDate) < new Date() && newOutstanding > 0) {
                          newStatus = 'Quá hạn';
                      }

                      await updateDoc(doc(window.db, `artifacts/${appId}/users/${window.userId}/debtRecords`, debtId), {
                          currentOutstanding: newOutstanding,
                          payments: updatedPayments,
                          status: newStatus,
                          updatedAt: new Date().toISOString()
                      });
                      window.showMessage('success', 'Thanh toán công nợ thành công!');
                  } catch (e) {
                      console.error("Error processing debt payment: ", e);
                      window.showMessage('error', `Lỗi khi thanh toán công nợ: ${e.message}`);
                  } finally {
                      window.hideLoadingOverlay();
                      window.closePayDebtModal();
                  }
              };
          }
      };

      window.closePayDebtModal = () => {
          const modal = document.getElementById('payDebtModal');
          if (modal) {
              modal.remove();
          }
      };

      // Initialize debt form on load
      document.addEventListener('DOMContentLoaded', () => {
          window.clearDebtForm();
      });

      // --- Debt Charts ---
      let debtChartInstance = null;
      let debtTrendChartInstance = null;

      window.renderDebtCharts = () => {
          // Chart 1: Total Debt by Supplier (Bar Chart)
          const debtBySupplierCtx = document.getElementById('debtBySupplierChart');
          const debtBySupplierChartNoDataMessage = document.getElementById('debtBySupplierChartNoDataMessage');

          if (debtBySupplierCtx) {
              if (debtChartInstance) {
                  debtChartInstance.destroy();
              }

              const supplierDebtMap = {};
              window.debtRecords.forEach(debt => {
                  if (debt.currentOutstanding > 0) { // Only count outstanding debt
                      const supplierName = window.getSupplierName(debt.supplierId);
                      supplierDebtMap[supplierName] = (supplierDebtMap[supplierName] || 0) + debt.currentOutstanding;
                  }
              });

              const labels = Object.keys(supplierDebtMap);
              const data = Object.values(supplierDebtMap);

              if (labels.length === 0) {
                  if (debtBySupplierChartNoDataMessage) debtBySupplierChartNoDataMessage.style.display = 'flex';
                  debtBySupplierCtx.style.display = 'none';
              } else {
                  if (debtBySupplierChartNoDataMessage) debtBySupplierChartNoDataMessage.style.display = 'none';
                  debtBySupplierCtx.style.display = 'block';
                  debtChartInstance = new Chart(debtBySupplierCtx, {
                      type: 'bar',
                      data: {
                          labels: labels,
                          datasets: [{
                              label: 'Tổng công nợ còn lại (VNĐ)',
                              data: data,
                              backgroundColor: 'rgba(139, 92, 246, 0.6)', // Primary color
                              borderColor: 'rgba(139, 92, 246, 1)',
                              borderWidth: 1
                          }]
                      },
                      options: {
                          responsive: true,
                          maintainAspectRatio: false,
                          scales: {
                              y: {
                                  beginAtZero: true,
                                  ticks: {
                                      callback: function(value) {
                                          return value.toLocaleString('vi-VN') + ' VNĐ';
                                      }
                                  }
                              }
                          },
                          plugins: {
                              tooltip: {
                                  callbacks: {
                                      label: function(context) {
                                          let label = context.dataset.label || '';
                                          if (label) {
                                              label += ': ';
                                          }
                                          if (context.parsed.y !== null) {
                                              label += context.parsed.y.toLocaleString('vi-VN') + ' VNĐ';
                                          }
                                          return label;
                                      }
                                  }
                              }
                          }
                      }
                  });
              }
          }

          // Chart 2: Debt Trend Over Time (Line Chart)
          const debtTrendCtx = document.getElementById('debtTrendChart');
          const debtTrendChartNoDataMessage = document.getElementById('debtTrendChartNoDataMessage');

          if (debtTrendCtx) {
              if (debtTrendChartInstance) {
                  debtTrendChartInstance.destroy();
              }

              // Aggregate debt over time (e.g., monthly)
              const monthlyDebt = {};
              window.debtRecords.forEach(debt => {
                  const date = new Date(debt.debtDate);
                  const monthYear = `${date.getFullYear()}-${(date.getMonth() + 1).toString().padStart(2, '0')}`;
                  monthlyDebt[monthYear] = (monthlyDebt[monthYear] || 0) + debt.currentOutstanding;
              });

              const sortedMonths = Object.keys(monthlyDebt).sort();
              const trendData = sortedMonths.map(month => monthlyDebt[month]);

              if (sortedMonths.length === 0) {
                  if (debtTrendChartNoDataMessage) debtTrendChartNoDataMessage.style.display = 'flex';
                  debtTrendCtx.style.display = 'none';
              } else {
                  if (debtTrendChartNoDataMessage) debtTrendChartNoDataMessage.style.display = 'none';
                  debtTrendCtx.style.display = 'block';
                  debtTrendChartInstance = new Chart(debtTrendCtx, {
                      type: 'line',
                      data: {
                          labels: sortedMonths,
                          datasets: [{
                              label: 'Tổng công nợ còn lại theo tháng (VNĐ)',
                              data: trendData,
                              borderColor: 'rgba(109, 40, 217, 1)', // Accent color
                              backgroundColor: 'rgba(109, 40, 217, 0.2)',
                              fill: true,
                              tension: 0.1
                          }]
                      },
                      options: {
                          responsive: true,
                          maintainAspectRatio: false,
                          scales: {
                              y: {
                                  beginAtZero: true,
                                  ticks: {
                                      callback: function(value) {
                                          return value.toLocaleString('vi-VN') + ' VNĐ';
                                      }
                                  }
                              }
                          },
                          plugins: {
                              tooltip: {
                                  callbacks: {
                                      label: function(context) {
                                          let label = context.dataset.label || '';
                                          if (label) {
                                              label += ': ';
                                          }
                                          if (context.parsed.y !== null) {
                                              label += context.parsed.y.toLocaleString('vi-VN') + ' VNĐ';
                                          }
                                          return label;
                                      }
                                  }
                              }
                          }
                      }
                  });
              }
          }
      };

      window.applyDebtFilters = () => {
          // This function would typically re-fetch or re-filter data based on selections
          // For now, it will just re-render charts with current data
          window.renderDebtList();
          window.renderDebtCharts();
          window.showMessage('info', 'Bộ lọc Công nợ đã được áp dụng (chức năng lọc dữ liệu sẽ được triển khai đầy đủ).');
      };


      // --- Tab: Price Comparison (So sánh giá) ---
      let priceComparisonChartInstance = null;

      window.renderPriceComparison = () => {
          const priceComparisonProductSelect = document.getElementById('priceComparisonProduct');
          const selectedProductId = priceComparisonProductSelect ? priceComparisonProductSelect.value : '';
          const priceComparisonTableBody = document.getElementById('priceComparisonTableBody');
          const priceComparisonChartCtx = document.getElementById('priceComparisonChart');
          const priceComparisonNoDataMessage = document.getElementById('priceComparisonChartNoDataMessage');

          if (priceComparisonChartInstance) {
              priceComparisonChartInstance.destroy();
          }
          if (priceComparisonTableBody) priceComparisonTableBody.innerHTML = '';
          if (priceComparisonChartCtx) priceComparisonChartCtx.style.display = 'none';
          if (priceComparisonNoDataMessage) priceComparisonNoDataMessage.style.display = 'flex';

          if (!selectedProductId) {
              if (priceComparisonTableBody) priceComparisonTableBody.innerHTML = `<tr><td colspan="3" class="text-center text-gray-500 italic">Vui lòng chọn một sản phẩm để so sánh giá.</td></tr>`;
              return;
          }

          const product = window.products.find(p => p.id === selectedProductId);
          if (!product || !product.supplierPrices || product.supplierPrices.length === 0) {
              if (priceComparisonTableBody) priceComparisonTableBody.innerHTML = `<tr><td colspan="3" class="text-center text-gray-500 italic">Không có dữ liệu giá cho sản phẩm này từ các nhà cung cấp.</td></tr>`;
              return;
          }

          const prices = product.supplierPrices.map(sp => ({
              supplierId: sp.supplierId, // Keep supplierId to get name
              supplierName: window.getSupplierName(sp.supplierId),
              price: sp.price
          })).sort((a, b) => a.price - b.price); // Sort by price ascending

          let lowestPrice = Infinity;
          if (prices.length > 0) {
              lowestPrice = prices[0].price;
          }

          const labels = [];
          const data = [];

          prices.forEach(item => {
              if (priceComparisonTableBody) {
                  const row = priceComparisonTableBody.insertRow();
                  const isLowest = item.price === lowestPrice;
                  row.className = isLowest ? 'highlighted-column' : ''; // Apply highlight
                  row.innerHTML = `
                      <td class="px-4 py-2">${item.supplierName}</td>
                      <td class="px-4 py-2">${item.price.toLocaleString('vi-VN')} VNĐ</td>
                      <td class="px-4 py-2">${isLowest ? '<span class="text-green-600 font-bold">Giá tốt nhất!</span>' : ''}</td>
                  `;
              }
              labels.push(item.supplierName);
              data.push(item.price);
          });

          if (labels.length > 0 && priceComparisonChartCtx) {
              if (priceComparisonNoDataMessage) priceComparisonNoDataMessage.style.display = 'none';
              priceComparisonChartCtx.style.display = 'block';
              priceComparisonChartInstance = new Chart(priceComparisonChartCtx, {
                  type: 'bar',
                  data: {
                      labels: labels,
                      datasets: [{
                          label: `Giá của ${product.name} (VNĐ)`,
                          data: data,
                          backgroundColor: labels.map((_, i) => prices[i].price === lowestPrice ? 'rgba(34, 197, 94, 0.6)' : 'rgba(99, 102, 241, 0.6)'), // Green for lowest, blue for others
                          borderColor: labels.map((_, i) => prices[i].price === lowestPrice ? 'rgba(34, 197, 94, 1)' : 'rgba(99, 102, 241, 1)'),
                          borderWidth: 1
                      }]
                  },
                  options: {
                      responsive: true,
                      maintainAspectRatio: false,
                      scales: {
                          y: {
                              beginAtZero: true,
                              ticks: {
                                  callback: function(value) {
                                      return value.toLocaleString('vi-VN') + ' VNĐ';
                                  }
                              }
                          }
                      },
                      plugins: {
                          tooltip: {
                              callbacks: {
                                  label: function(context) {
                                      let label = context.dataset.label || '';
                                      if (label) {
                                          label += ': ';
                                      }
                                      if (context.parsed.y !== null) {
                                          label += context.parsed.y.toLocaleString('vi-VN') + ' VNĐ';
                                      }
                                      return label;
                                  }
                              }
                          }
                      }
                  }
              });
          }
      };

      // Function to update product's supplierPrices array
      window.updateProductSupplierPrice = async (productId, supplierId, price) => {
          window.showLoadingOverlay();
          try {
              const productRef = doc(window.db, `artifacts/${appId}/users/${window.userId}/products`, productId);
              const productDoc = await getDoc(productRef);
              if (productDoc.exists()) {
                  const productData = productDoc.data();
                  let supplierPrices = productData.supplierPrices || [];
                  const existingIndex = supplierPrices.findIndex(sp => sp.supplierId === supplierId);

                  if (existingIndex > -1) {
                      supplierPrices[existingIndex] = { supplierId, price, lastUpdated: new Date().toISOString() };
                  } else {
                      supplierPrices.push({ supplierId, price, lastUpdated: new Date().toISOString() });
                  }
                  await updateDoc(productRef, { supplierPrices: supplierPrices });
                  window.showMessage('success', 'Giá sản phẩm từ nhà cung cấp đã được cập nhật!');
              } else {
                  window.showMessage('error', 'Không tìm thấy sản phẩm.');
              }
          } catch (e) {
              console.error("Error updating product supplier price:", e);
              window.showMessage('error', `Lỗi khi cập nhật giá: ${e.message}`);
          } finally {
              window.hideLoadingOverlay();
          }
      };

      // Modal for quick price update for a product from a supplier
      window.openQuickPriceUpdateModal = () => {
          const modalHtml = `
              <div id="quickPriceUpdateModal" class="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-[7000]">
                  <div class="bg-white p-6 rounded-lg shadow-xl w-full max-w-md">
                      <h3 class="text-xl font-semibold mb-4 text-gray-700">Cập nhật giá nhanh</h3>
                      <div class="form-group mb-4">
                          <label for="quickPriceProduct">Sản phẩm:</label>
                          <div class="flex items-center">
                              <select id="quickPriceProduct" data-type="product" class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white"></select>
                              <button type="button" class="plus-button ml-2" onclick="window.openQuickAddModal('product-quick', 'quickPriceProduct')">+</button>
                          </div>
                      </div>
                      <div class="form-group mb-4">
                          <label for="quickPriceSupplier">Nhà cung cấp:</label>
                          <div class="flex items-center">
                              <select id="quickPriceSupplier" data-type="supplier" class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white"></select>
                              <button type="button" class="plus-button ml-2" onclick="window.openQuickAddModal('supplier-quick', 'quickPriceSupplier')">+</button>
                          </div>
                      </div>
                      <div class="form-group mb-4">
                          <label for="quickPriceAmount">Giá:</label>
                          <input type="number" id="quickPriceAmount" placeholder="Giá sản phẩm" class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500">
                      </div>
                      <div class="flex justify-end gap-4">
                          <button type="button" class="btn btn-secondary" onclick="window.closeQuickPriceUpdateModal()">Hủy</button>
                          <button type="button" class="btn btn-primary" id="saveQuickPriceBtn">Lưu</button>
                      </div>
                  </div>
              </div>
          `;
          document.body.insertAdjacentHTML('beforeend', modalHtml);

          // Populate dropdowns in the modal
          const quickPriceProductSelect = document.getElementById('quickPriceProduct');
          const quickPriceSupplierSelect = document.getElementById('quickPriceSupplier');
          if (quickPriceProductSelect) window.populateProductDropdownsForModal(quickPriceProductSelect); // Use a dedicated function to ensure fresh options
          if (quickPriceSupplierSelect) window.populateSupplierDropdownsForModal(quickPriceSupplierSelect); // Use a dedicated function

          const saveQuickPriceBtn = document.getElementById('saveQuickPriceBtn');
          if (saveQuickPriceBtn) {
              saveQuickPriceBtn.onclick = async () => {
                  const productId = quickPriceProductSelect ? quickPriceProductSelect.value : '';
                  const supplierId = quickPriceSupplierSelect ? quickPriceSupplierSelect.value : '';
                  const quickPriceAmountInput = document.getElementById('quickPriceAmount');
                  const price = parseFloat(quickPriceAmountInput ? quickPriceAmountInput.value : '');

                  if (!productId || !supplierId || isNaN(price) || price <= 0) {
                      window.showMessage('error', 'Vui lòng chọn sản phẩm, nhà cung cấp và nhập giá hợp lệ.');
                      return;
                  }
                  await window.updateProductSupplierPrice(productId, supplierId, price);
                  window.closeQuickPriceUpdateModal();
              };
          }
      };

      window.closeQuickPriceUpdateModal = () => {
          const modal = document.getElementById('quickPriceUpdateModal');
          if (modal) {
              modal.remove();
          }
      };

      // Helper functions for populating modals' dropdowns (to ensure they are fresh)
      window.populateProductDropdownsForModal = (selectElement) => {
          selectElement.innerHTML = '<option value="">Chọn Sản phẩm</option>';
          window.products.forEach(product => {
              const option = document.createElement('option');
              option.value = product.id;
              option.textContent = product.name;
              selectElement.appendChild(option);
          });
      };

      window.populateSupplierDropdownsForModal = (selectElement) => {
          selectElement.innerHTML = '<option value="">Chọn Nhà cung cấp</option>';
          window.suppliers.forEach(supplier => {
              const option = document.createElement('option');
              option.value = supplier.id;
              option.textContent = supplier.name;
              selectElement.appendChild(option);
          });
      };


      // --- Tab: Dashboard (Tổng quan) ---
      let dashboardSupplierPerformanceChartInstance = null;
      let dashboardSupplierSpendingChartInstance = null;

      window.renderDashboardCharts = () => {
          // Chart 1: Supplier Performance (Spider Chart)
          const performanceCtx = document.getElementById('dashboardSupplierPerformanceChart');
          const dashboardPerformanceChartNoDataMessage = document.getElementById('dashboardPerformanceChartNoDataMessage');

          if (performanceCtx) {
              if (dashboardSupplierPerformanceChartInstance) {
                  dashboardSupplierPerformanceChartInstance.destroy();
              }

              // Placeholder for supplier performance data (you'd need a rating system)
              // For demonstration, let's assume some dummy data or average from orders
              // Criteria: Quality, Delivery Speed, Price Competitiveness, Support, Responsiveness
              const performanceData = window.suppliers.map(supplier => {
                  // Dummy data for now, replace with actual rating logic
                  // Example: Calculate average on-time delivery from orders
                  const supplierOrders = window.orders.filter(order => order.supplierId === supplier.id);
                  const totalOrders = supplierOrders.length;
                  const onTimeOrders = supplierOrders.filter(order => {
                      return order.deliveryDate && new Date(order.deliveryDate) <= new Date(order.orderDate).setDate(new Date(order.orderDate).getDate() + 7); // Delivered within 7 days
                  }).length;
                  const onTimeRate = totalOrders > 0 ? (onTimeOrders / totalOrders) * 5 : 3; // Scale to 1-5

                  // Dummy values for other criteria
                  const quality = Math.floor(Math.random() * 2) + 3; // Mostly 3-5
                  const priceCompetitiveness = Math.floor(Math.random() * 2) + 3; // Mostly 3-5
                  const customerSupport = Math.floor(Math.random() * 2) + 3; // Mostly 3-5
                  const responsiveness = Math.floor(Math.random() * 2) + 3; // Mostly 3-5

                  return {
                      name: supplier.name,
                      quality: quality,
                      deliverySpeed: onTimeRate,
                      priceCompetitiveness: priceCompetitiveness,
                      customerSupport: customerSupport,
                      responsiveness: responsiveness
                  };
              });

              const labels = ['Chất lượng', 'Tốc độ giao hàng', 'Giá cả', 'Hỗ trợ khách hàng', 'Phản hồi'];
              const datasets = performanceData.map((data, index) => ({
                  label: data.name,
                  data: [data.quality, data.deliverySpeed, data.priceCompetitiveness, data.customerSupport, data.responsiveness],
                  backgroundColor: `rgba(${Math.floor(Math.random() * 255)}, ${Math.floor(Math.random() * 255)}, ${Math.floor(Math.random() * 255)}, 0.4)`,
                  borderColor: `rgba(${Math.floor(Math.random() * 255)}, ${Math.floor(Math.random() * 255)}, ${Math.floor(Math.random() * 255)}, 1)`,
                  borderWidth: 1
              }));

              if (datasets.length === 0 || datasets.every(ds => ds.data.every(val => val === 0))) {
                  if (dashboardPerformanceChartNoDataMessage) dashboardPerformanceChartNoDataMessage.style.display = 'flex';
                  performanceCtx.style.display = 'none';
              } else {
                  if (dashboardPerformanceChartNoDataMessage) dashboardPerformanceChartNoDataMessage.style.display = 'none';
                  performanceCtx.style.display = 'block';
                  dashboardSupplierPerformanceChartInstance = new Chart(performanceCtx, {
                      type: 'radar',
                      data: {
                          labels: labels,
                          datasets: datasets
                      },
                      options: {
                          responsive: true,
                          maintainAspectRatio: false,
                          elements: {
                              line: {
                                  borderWidth: 3
                              }
                          },
                          scales: {
                              r: {
                                  angleLines: {
                                      display: false
                                  },
                                  suggestedMin: 0,
                                  suggestedMax: 5,
                                  ticks: {
                                      stepSize: 1
                                  }
                              }
                          },
                          plugins: {
                              legend: {
                                  position: 'top',
                              },
                              title: {
                                  display: true,
                                  text: 'Đánh giá hiệu suất nhà cung cấp'
                              }
                          }
                      }
                  });
              }
          }

          // Chart 2: Total Spending by Supplier (Bar Chart)
          const spendingCtx = document.getElementById('dashboardSupplierSpendingChart');
          const dashboardChartNoDataMessage = document.getElementById('dashboardChartNoDataMessage');

          if (spendingCtx) {
              if (dashboardSupplierSpendingChartInstance) {
                  dashboardSupplierSpendingChartInstance.destroy();
              }

              const supplierSpendingMap = {};
              window.orders.forEach(order => {
                  const supplierName = window.getSupplierName(order.supplierId);
                  supplierSpendingMap[supplierName] = (supplierSpendingMap[supplierName] || 0) + (order.totalAmount || 0);
              });

              const labels = Object.keys(supplierSpendingMap);
              const data = Object.values(supplierSpendingMap);

              if (labels.length === 0) {
                  if (dashboardChartNoDataMessage) dashboardChartNoDataMessage.style.display = 'flex';
                  spendingCtx.style.display = 'none';
              } else {
                  if (dashboardChartNoDataMessage) dashboardChartNoDataMessage.style.display = 'none';
                  spendingCtx.style.display = 'block';
                  dashboardSupplierSpendingChartInstance = new Chart(spendingCtx, {
                      type: 'bar',
                      data: {
                          labels: labels,
                          datasets: [{
                              label: 'Tổng chi tiêu (VNĐ)',
                              data: data,
                              backgroundColor: 'rgba(102, 108, 255, 0.6)', // A different shade of blue/purple
                              borderColor: 'rgba(102, 108, 255, 1)',
                              borderWidth: 1
                          }]
                      },
                      options: {
                          responsive: true,
                          maintainAspectRatio: false,
                          scales: {
                              y: {
                                  beginAtZero: true,
                                  ticks: {
                                      callback: function(value) {
                                          return value.toLocaleString('vi-VN') + ' VNĐ';
                                      }
                                  }
                              }
                          },
                          plugins: {
                              tooltip: {
                                  callbacks: {
                                      label: function(context) {
                                          let label = context.dataset.label || '';
                                          if (label) {
                                              label += ': ';
                                          }
                                          if (context.parsed.y !== null) {
                                              label += context.parsed.y.toLocaleString('vi-VN') + ' VNĐ';
                                          }
                                          return label;
                                      }
                                  }
                              }
                          }
                      }
                  });
              }
          }

          // Update Scoreboard table (dummy data for now)
          const nccScoreboardBody = document.querySelector('#nccScoreboard tbody');
          if (nccScoreboardBody) {
              nccScoreboardBody.innerHTML = '';
              if (window.suppliers.length === 0) {
                  nccScoreboardBody.innerHTML = `<tr><td colspan="3" class="text-center text-gray-500 italic">Chưa có dữ liệu nhà cung cấp.</td></tr>`;
              } else {
                  window.suppliers.forEach(supplier => {
                      const row = nccScoreboardBody.insertRow();
                      // Dummy data for score and on-time rate
                      const score = (Math.random() * 100).toFixed(1);
                      const onTimeRate = (Math.random() * 100).toFixed(1);
                      row.innerHTML = `
                          <td class="px-4 py-2">${supplier.name}</td>
                          <td class="px-4 py-2">${score}</td>
                          <td class="px-4 py-2">${onTimeRate}%</td>
                      `;
                  });
              }
          }
      };

      window.applyDashboardFilters = () => {
          // This function would typically re-fetch or re-filter data based on selections
          // For now, it will just re-render charts with current data
          window.renderDashboardCharts();
          window.showMessage('info', 'Bộ lọc Dashboard đã được áp dụng (chức năng lọc dữ liệu sẽ được triển khai đầy đủ).');
      };

      // --- Tab: Settings (Cài đặt) ---
      window.renderSettingsTab = () => {
          // No specific rendering needed beyond what's in HTML, but can add dynamic content here if needed.
          console.log("Settings tab rendered.");
          // Set initial theme based on local storage or default
          const savedTheme = localStorage.getItem('theme') || 'theme-light';
          document.body.className = savedTheme;
          const themeRadios = document.querySelectorAll('input[name="theme-option"]');
          themeRadios.forEach(radio => {
              if (radio.value === savedTheme) {
                  radio.checked = true;
              }
          });
      };

      window.changeTheme = (theme) => {
          document.body.className = theme;
          localStorage.setItem('theme', theme);
          window.showMessage('info', `Đã chuyển sang giao diện ${theme.replace('theme-', '')}`);
      };


      // --- Event Listeners for Quick Add Buttons (from original HTML) ---
      document.addEventListener('DOMContentLoaded', () => {
          // Product tab quick add buttons
          const addQuickProductUnitBtn = document.getElementById('addQuickProductUnitBtn');
          if (addQuickProductUnitBtn) addQuickProductUnitBtn.addEventListener('click', () => window.openQuickAddModal('unit', 'productUnit'));
          const addQuickProductCategoryBtn = document.getElementById('addQuickProductCategoryBtn');
          if (addQuickProductCategoryBtn) addQuickProductCategoryBtn.addEventListener('click', () => window.openQuickAddModal('category', 'productCategory'));

          // Order History tab quick add buttons
          const addOrderSupplierBtn = document.getElementById('addOrderSupplierBtn');
          if (addOrderSupplierBtn) addOrderSupplierBtn.addEventListener('click', () => window.openQuickAddModal('supplier-quick', 'orderSupplier'));
          const addOrderProductQuickBtn = document.getElementById('addOrderProductQuickBtn');
          if (addOrderProductQuickBtn) addOrderProductQuickBtn.addEventListener('click', () => window.openQuickAddModal('product-quick', 'orderProduct-0')); // Assumes first item is always there

          // Debt Management tab quick add buttons
          const addDebtSupplierBtn = document.getElementById('addDebtSupplierBtn');
          if (addDebtSupplierBtn) addDebtSupplierBtn.addEventListener('click', () => window.openQuickAddModal('supplier-quick', 'debtSupplier'));
          const addDebtBankBtn = document.getElementById('addDebtBankBtn');
          if (addDebtBankBtn) addDebtBankBtn.addEventListener('click', () => window.openQuickAddModal('bank', 'debtPaymentMethod')); // Assuming payment method might be a bank

          // Price Comparison tab quick add buttons - these are in the modal, handled by openQuickPriceUpdateModal
          const updatePriceBtn = document.getElementById('updatePriceBtn');
          if (updatePriceBtn) updatePriceBtn.addEventListener('click', window.openQuickPriceUpdateModal); // Open modal for quick price update

          // Inventory tab quick add buttons
          const addInventorySenderBtn = document.getElementById('addInventorySenderBtn');
          if (addInventorySenderBtn) addInventorySenderBtn.addEventListener('click', () => window.openQuickAddModal('entity-sender', 'inventoryMovementSender'));
          const addInventoryRecipientBtn = document.getElementById('addInventoryRecipientBtn');
          if (addInventoryRecipientBtn) addInventoryRecipientBtn.addEventListener('click', () => window.openQuickAddModal('entity-recipient', 'inventoryMovementRecipient'));


          // Initial population of all dropdowns when data is loaded
          // These will be called by onSnapshot listeners, but good to have a manual call too.
          window.populateUnitDropdowns();
          window.populateCategoryDropdowns();
          window.populateSupplierDropdowns();
          window.populateProductDropdowns();
          window.populateBankDropdowns();
          window.populateEntityDropdowns();
          window.populatePaymentMethodDropdowns();

          // Initial render of lists and charts for the default active tab
          window.renderProductList();
          window.renderSupplierList();
          window.renderOrderHistory();
          window.renderInventoryList();
          window.renderDebtList();
          window.renderDashboardCharts();
          window.renderPriceComparison(); // Initial render for price comparison tab
          window.renderSettingsTab(); // Initial render for settings tab
      });

      // Placeholder for logoutUser function (for future authentication integration)
      window.logoutUser = () => {
          window.showMessage('info', 'Chức năng đăng xuất sẽ được triển khai với hệ thống xác thực Firebase.');
          // In a real Firebase app, you would call firebase.auth().signOut();
      }
    </script>
    <style>
      /* Custom CSS for pastel theme, fonts, and animations */
      :root {
          --bg-color: #f8fafc; /* Slate 50 */
          --text-color: #334155; /* Slate 700 */
          --text-color-secondary: #64748b; /* Slate 500 */
          --primary-color: #8b5cf6; /* Violet 500 */
          --primary-color-dark: #7c3aed; /* Violet 600 */
          --accent-color: #6d28d9; /* Violet 700 */
          --accent-color-light: #ede9fe; /* Violet 100 */
          --card-bg-color: #ffffff;
          --hover-bg-color: #f1f5f9; /* Slate 100 */
          --hover-text-color: #475569; /* Slate 600 */
          --border-color: #cbd5e1; /* Slate 300 */
          --input-bg-color: #f8fafc; /* Slate 50 */
          --button-text-color: #ffffff;
          --table-header-bg: #f1f5f9; /* Slate 100 */
          --table-row-hover-bg: #f8fafc; /* Slate 50 */
      }

      body {
          font-family: 'Inter', sans-serif;
          background-color: var(--bg-color);
          color: var(--text-color);
          margin: 0;
          padding: 0;
          display: flex; /* Use flexbox for sidebar layout */
          min-height: 100vh;
          overflow-x: hidden; /* Prevent horizontal scroll */
      }

      /* Sidebar styling */
      .sidebar {
          width: 280px; /* Fixed width for sidebar */
          background-color: var(--card-bg-color);
          border-right: 1px solid #e2e8f0;
          box-shadow: 2px 0 5px rgba(0, 0, 0, 0.05);
          padding: 1.5rem 1rem;
          display: flex;
          flex-direction: column;
          position: sticky; /* Make sidebar sticky */
          top: 0;
          height: 100vh; /* Full height */
          overflow-y: auto; /* Scrollable if content overflows */
          flex-shrink: 0; /* Prevent sidebar from shrinking */
      }

      .sidebar-header {
          display: flex;
          align-items: center;
          margin-bottom: 2rem;
          padding: 0 0.5rem;
      }

      .sidebar-header .logo {
          height: 48px; /* Larger logo */
          width: auto;
          margin-right: 0.75rem;
      }

      .sidebar-header h1 {
          font-size: 1.5rem; /* Larger title */
          font-weight: 700;
          color: var(--primary-color);
          line-height: 1.2;
      }

      .sidebar-nav {
          flex-grow: 1; /* Allow nav to take available space */
      }

      .sidebar-button {
          display: flex;
          align-items: center;
          width: 100%; /* Ensure button takes full width */
          padding: 0.75rem 1rem;
          margin-bottom: 0.5rem;
          cursor: pointer;
          border-radius: 0.75rem; /* Rounded corners */
          font-weight: 500;
          color: var(--text-color-secondary); /* Gray text for inactive buttons */
          transition: all 0.3s ease-in-out;
          border: none; /* Remove default button border */
          background-color: transparent; /* Transparent background */
      }

      .sidebar-button:hover {
          background-color: var(--hover-bg-color);
          color: var(--hover-text-color);
      }

      .sidebar-button.active {
          background-color: var(--accent-color-light); /* Light purple background for active button */
          color: var(--accent-color); /* Deep purple for active button */
          font-weight: 600;
      }

      .sidebar-button svg {
          margin-right: 0.75rem;
          width: 20px;
          height: 20px;
      }

      /* Main content area */
      .main-content {
          flex-grow: 1;
          padding: 2rem;
          background-color: var(--bg-color);
          overflow-y: auto; /* Allow main content to scroll */
      }

      /* Card styling */
      .card {
          background-color: var(--card-bg-color);
          border-radius: 0.75rem; /* Rounded corners */
          box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
          padding: 1.5rem;
          margin-bottom: 1.5rem;
      }

      /* Form and input styling */
      .form-group label {
          display: block;
          margin-bottom: 0.5rem;
          font-weight: 500;
          color: var(--text-color);
      }

      .form-group input,
      .form-group select,
      .form-group textarea {
          width: 100%;
          padding: 0.75rem;
          border: 1px solid var(--border-color);
          border-radius: 0.5rem;
          background-color: var(--input-bg-color);
          font-family: 'Roboto', sans-serif;
          transition: border-color 0.2s ease-in-out, box-shadow 0.2s ease-in-out;
      }

      .form-group input:focus,
      .form-group select:focus,
      .form-group textarea:focus {
          outline: none;
          border-color: var(--primary-color); /* Purple focus border */
          box-shadow: 0 0 0 3px rgba(139, 92, 246, 0.2);
      }

      .btn {
          display: inline-flex;
          align-items: center;
          justify-content: center;
          padding: 0.75rem 1.5rem;
          border-radius: 0.5rem;
          font-weight: 600;
          cursor: pointer;
          transition: background-color 0.2s ease-in-out, box-shadow 0.2s ease-in-out;
          border: none;
      }

      .btn-primary {
          background-color: var(--primary-color); /* Purple */
          color: var(--button-text-color);
      }

      .btn-primary:hover {
          background-color: var(--primary-color-dark); /* Darker purple */
          box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
      }

      .btn-secondary {
          background-color: var(--secondary-color, #e2e8f0); /* Light gray */
          color: var(--text-color);
      }

      .btn-secondary:hover {
          background-color: var(--secondary-color-dark, #cbd5e1);
      }

      /* Table styling */
      .data-table {
          width: 100%;
          border-collapse: collapse;
          margin-top: 1rem;
      }

      .data-table th,
      .data-table td {
          padding: 0.75rem;
          text-align: left;
          border-bottom: 1px solid var(--border-color);
      }

      .data-table th {
          background-color: var(--table-header-bg);
          font-weight: 600;
          color: var(--text-color);
          text-transform: uppercase;
          font-size: 0.875rem;
      }

      .data-table tbody tr:hover {
          background-color: var(--table-row-hover-bg);
      }

      /* Tab content styling */
      .tab-content {
          display: none; /* Hidden by default */
          animation: fadeIn 0.5s ease-out; /* Fade in animation */
      }

      .tab-content.active {
          display: block;
      }

      /* Responsive adjustments */
      @media (max-width: 1024px) { /* Adjust sidebar for tablets */
          .sidebar {
              width: 80px; /* Smaller width for icons only */
              padding: 1.5rem 0.5rem;
          }
          .sidebar-header h1 {
              display: none; /* Hide title */
          }
          .sidebar-button span {
              display: none; /* Hide button text */
          }
          .sidebar-button {
              justify-content: center; /* Center icons */
              padding: 0.75rem;
          }
          .sidebar-button svg {
              margin-right: 0;
          }
          .main-content {
              padding: 1.5rem;
          }
      }

      @media (max-width: 768px) { /* Mobile */
          body {
              flex-direction: column; /* Stack sidebar and content */
          }
          .sidebar {
              width: 100%;
              height: auto;
              position: relative;
              box-shadow: 0 2px 5px rgba(0, 0, 0, 0.05);
              border-right: none;
              border-bottom: 1px solid #e2e8f0;
              flex-direction: row; /* Horizontal for mobile */
              justify-content: space-around;
              padding: 1rem 0.5rem;
              overflow-x: auto; /* Allow horizontal scroll for tabs if many */
              -webkit-overflow-scrolling: touch; /* Smooth scrolling on iOS */
          }
          .sidebar-header {
              display: none; /* Hide header on mobile */
          }
          .sidebar-nav {
              display: flex;
              flex-direction: row;
              width: 100%;
              justify-content: space-around;
          }
          .sidebar-button {
              flex-direction: column; /* Stack icon and text */
              font-size: 0.75rem;
              padding: 0.5rem;
              margin-bottom: 0;
              margin-right: 0.25rem;
              margin-left: 0.25rem;
              border-radius: 0.5rem;
          }
          .sidebar-button span {
              display: block; /* Show text again, but smaller */
              margin-top: 0.25rem;
              white-space: nowrap;
          }
          .sidebar-button svg {
              margin-right: 0;
              margin-bottom: 0.25rem;
          }
          .main-content {
              padding: 1rem;
          }
          .modal-content-grid {
              grid-template-columns: 1fr; /* Stack columns on mobile */
          }
          .modal-content-grid .full-width {
              grid-column: span 1;
          }
      }

      /* Loading animation */
      .loading-overlay {
          position: fixed;
          top: 0;
          left: 0;
          width: 100%;
          height: 100%;
          background-color: rgba(255, 255, 255, 0.8);
          display: flex;
          justify-content: center;
          align-items: center;
          z-index: 9999;
          transition: opacity 0.3s ease-in-out;
          opacity: 0;
          visibility: hidden;
      }

      .loading-overlay.visible {
          opacity: 1;
          visibility: visible;
      }

      .spinner {
          border: 4px solid rgba(0, 0, 0, 0.1);
          border-left-color: var(--primary-color); /* Purple spinner */
          border-radius: 50%;
          width: 40px;
          height: 40px;
          animation: spin 1s linear infinite;
      }

      @keyframes spin {
          0% { transform: rotate(0deg); }
          100% { transform: rotate(360deg); }
      }

      /* Fade in animation for tab content */
      @keyframes fadeIn {
          from { opacity: 0; transform: translateY(10px); }
          to { opacity: 1; transform: translateY(0); }
      }

      /* Specific colors for status/alerts */
      .status-overdue {
          color: #ef4444; /* Red */
          font-weight: 600;
      }
      .status-due-soon {
          color: #f97316; /* Orange */
          font-weight: 600;
      }
      .status-on-time {
          color: #22c55e; /* Green */
          font-weight: 600;
      }

      /* Styles for dynamic item rows in inventory form */
      .inventory-item-row {
          display: flex;
          gap: 0.75rem; /* Tailwind gap-3 */
          margin-bottom: 0.75rem; /* Tailwind mb-3 */
          align-items: flex-end; /* Align items to the bottom */
      }

      .inventory-item-row .form-group {
          flex: 1; /* Make form groups take equal width */
          margin-bottom: 0; /* Remove bottom margin from inner form groups */
      }

      .inventory-item-row .form-group input,
      .inventory-item-row .form-group span {
          width: 100%;
      }

      .inventory-item-row .remove-item-btn {
          background-color: #ef4444; /* Red 500 */
          color: white;
          padding: 0.5rem 0.75rem; /* px-3 py-2 */
          border-radius: 0.5rem; /* rounded-md */
          font-weight: 600;
          cursor: pointer;
          transition: background-color 0.2s ease-in-out;
          border: none;
          height: 42px; /* Match input height */
          display: flex;
          align-items: center;
          justify-content: center;
      }

      .inventory-item-row .remove-item-btn:hover {
          background-color: #dc2626; /* Red 600 */
      }

      /* Modal specific styling for better layout */
      .modal-content-grid {
          display: grid;
          grid-template-columns: 1fr 1fr;
          gap: 1rem; /* gap-4 */
      }

      .modal-content-grid .full-width {
          grid-column: span 2;
      }

      /* Plus button styling */
      .plus-button {
          background-color: var(--accent-color); /* Deep purple */
          color: white;
          border-radius: 9999px; /* Full rounded */
          width: 28px; /* w-7 */
          height: 28px; /* h-7 */
          display: flex;
          align-items: center;
          justify-content: center;
          font-size: 1.25rem; /* text-xl */
          line-height: 1;
          cursor: pointer;
          transition: background-color 0.2s ease-in-out, transform 0.2s ease-in-out;
          margin-left: 0.5rem; /* ml-2 */
          flex-shrink: 0; /* Prevent shrinking */
      }

      .plus-button:hover {
          background-color: var(--accent-color-dark, #5b21b6); /* Darker purple */
          transform: scale(1.1);
      }

      /* Autocomplete suggestions dropdown */
      .autocomplete-suggestions {
          position: absolute;
          background-color: white;
          border: 1px solid var(--border-color); /* Tailwind gray-200 */
          border-radius: 0.5rem; /* Tailwind rounded-md */
          max-height: 200px;
          overflow-y: auto;
          z-index: 10;
          width: calc(100% - 36px); /* Adjust width to account for plus button */
          box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06); /* Tailwind shadow-md */
          left: 0;
          top: 100%;
      }
      .autocomplete-suggestions div {
          padding: 0.5rem 0.75rem;
          cursor: pointer;
      }
      .autocomplete-suggestions div:hover {
          background-color: var(--hover-bg-color); /* Tailwind gray-100 */
      }
      .autocomplete-container {
          position: relative; /* Needed for absolute positioning of suggestions */
      }

      /* Message box styling */
      .message-box {
          position: fixed;
          top: 1rem;
          right: 1rem;
          padding: 1rem;
          border-radius: 0.5rem;
          box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
          z-index: 6000;
          display: flex;
          align-items: center;
          transition: opacity 0.3s ease-in-out, transform 0.3s ease-in-out;
          opacity: 0;
          visibility: hidden;
      }

      .message-box.show {
          opacity: 1;
          transform: translateY(0);
          visibility: visible;
      }

      .message-box.info { background-color: #3b82f6; color: white; } /* Blue 500 */
      .message-box.success { background-color: #22c55e; color: white; } /* Green 500 */
      .message-box.error { background-color: #ef4444; color: white; } /* Red 500 */

      .message-box button {
          background: none;
          border: none;
          color: inherit;
          font-size: 1.25rem;
          margin-left: 1rem;
          cursor: pointer;
      }

      /* Theme customization */
      body.theme-light {
          --bg-color: #f8fafc;
          --text-color: #334155;
          --text-color-secondary: #64748b;
          --primary-color: #8b5cf6;
          --primary-color-dark: #7c3aed;
          --accent-color: #6d28d9;
          --accent-color-light: #ede9fe;
          --card-bg-color: #ffffff;
          --hover-bg-color: #f1f5f9;
          --hover-text-color: #475569;
          --border-color: #cbd5e1;
          --input-bg-color: #f8fafc;
          --button-text-color: #ffffff;
          --table-header-bg: #f1f5f9;
          --table-row-hover-bg: #f8fafc;
      }

      body.theme-dark {
          --bg-color: #1a202c; /* Dark background */
          --text-color: #e2e8f0; /* Light text */
          --text-color-secondary: #a0aec0;
          --primary-color: #667eea; /* Blue-violet */
          --primary-color-dark: #5a67d8;
          --accent-color: #4c51bf;
          --accent-color-light: #2d3748;
          --card-bg-color: #2d3748; /* Darker card */
          --hover-bg-color: #4a5568;
          --hover-text-color: #e2e8f0;
          --border-color: #4a5568;
          --input-bg-color: #2d3748;
          --button-text-color: #ffffff;
          --table-header-bg: #4a5568;
          --table-row-hover-bg: #2d3748;
      }

      body.theme-pastel {
          --bg-color: #f0f8ff; /* AliceBlue */
          --text-color: #5f6b7c; /* Desaturated blue */
          --text-color-secondary: #9bb0c9;
          --primary-color: #a78bfa; /* Light violet */
          --primary-color-dark: #8b5cf6;
          --accent-color: #c4b5fd;
          --accent-color-light: #eef2ff;
          --card-bg-color: #ffffff;
          --hover-bg-color: #e0f2f7; /* Light cyan */
          --hover-text-color: #7c8b9f;
          --border-color: #dbe4ed;
          --input-bg-color: #f0f8ff;
          --button-text-color: #ffffff;
          --table-header-bg: #e0f2f7;
          --table-row-hover-bg: #f0f8ff;
      }

      .highlighted-column {
          background-color: #d1e7dd; /* Light green for highlight */
          transition: background-color 0.3s ease-in-out;
      }
    </style>
  </head>
  <body>
    <!-- Loading Overlay -->
    <div id="loadingOverlay" class="loading-overlay">
      <div class="spinner"></div>
    </div>

    <!-- Message Box Container -->
    <div id="messageContainer" class="message-box"></div>

    <!-- Login Modal (Hidden by default for now) -->
    <div
      id="loginModal"
      class="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-50 hidden"
    >
      <div class="bg-white p-6 rounded-lg shadow-xl w-full max-w-md">
        <h3 class="text-xl font-semibold mb-4 text-gray-700">Đăng nhập</h3>
        <form id="loginForm">
          <div class="form-group mb-4">
            <label for="loginUsername">Tên đăng nhập:</label>
            <input type="text" id="loginUsername" required />
          </div>
          <div class="form-group mb-4">
            <label for="loginPassword">Mật khẩu:</label>
            <input type="password" id="loginPassword" required />
          </div>
          <div class="flex justify-end gap-4">
            <button
              type="button"
              class="btn btn-secondary"
              onclick="openRegisterModal()"
            >
              Đăng ký
            </button>
            <button type="submit" class="btn btn-primary">Đăng nhập</button>
          </div>
        </form>
      </div>
    </div>

    <!-- Register Modal (Hidden by default for now) -->
    <div
      id="registerModal"
      class="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-50 hidden"
    >
      <div class="bg-white p-6 rounded-lg shadow-xl w-full max-w-md">
        <h3 class="text-xl font-semibold mb-4 text-gray-700">
          Đăng ký tài khoản mới
        </h3>
        <form id="registerForm">
          <div class="form-group mb-4">
            <label for="registerUsername">Tên đăng nhập:</label>
            <input type="text" id="registerUsername" required />
          </div>
          <div class="form-group mb-4">
            <label for="registerPassword">Mật khẩu:</label>
            <input type="password" id="registerPassword" required />
          </div>
          <div class="form-group mb-4">
            <label for="registerRole">Vai trò:</label>
            <select id="registerRole" required>
              <option value="staff">Nhân viên</option>
              <option value="manager">Quản lý</option>
              <option value="admin">Admin</option>
            </select>
          </div>
          <div class="flex justify-end gap-4">
            <button
              type="button"
              class="btn btn-secondary"
              onclick="closeRegisterModal()"
            >
              Hủy
            </button>
            <button type="submit" class="btn btn-primary">Đăng ký</button>
          </div>
        </form>
      </div>
    </div>

    <!-- Sidebar -->
    <aside class="sidebar">
      <div class="sidebar-header">
        <img
          id="appLogo"
          src="https://placehold.co/48x48/6d28d9/ffffff?text=LOGO"
          alt="Company Logo"
          class="logo rounded-full"
        />
        <h1 id="appTitle" class="text-gray-800">Quản Lý NCC & SP</h1>
      </div>

      <nav class="sidebar-nav">
        <button class="sidebar-button active" data-tab="dashboard">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
            stroke-width="2"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M3 12l2-2m0 0l7-7 7 7M5 10v10a1 1 0 001 1h3m10-11l2 2m-2 2v10a1 1 0 01-1 1h-3m-6 0a1 1 0 001-1v-4a1 1 0 011-1h2a1 1 0 011 1v4a1 1 0 001 1m-6 0h6"
            />
          </svg>
          <span>Dashboard</span>
        </button>
        <button class="sidebar-button" data-tab="priceComparison">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
            stroke-width="2"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M9 14V9m0 0a3 3 0 01-3-3V4a1 1 0 011-1h2m0 0h4m-4 0a3 3 0 013 3v2m0 0V9m0 0a3 3 0 003 3v2m0 0V9m0 0a3 3 0 01-3 3v2m0 0V9"
            />
          </svg>
          <span>So sánh giá</span>
        </button>
        <button class="sidebar-button" data-tab="inventory">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
            stroke-width="2"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M20 7l-8-4-8 4m16 0l-8 4m8-4v10l-8 4m0-10L4 7m8 4v10m-8-4l8 4m-8-4V7"
            />
          </svg>
          <span>Tồn kho</span>
        </button>
        <button class="sidebar-button" data-tab="debtManagement">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
            stroke-width="2"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M17 9V7a2 2 0 00-2-2H5a2 2 0 00-2 2v6a2 2 0 002 2h2m2 4h10a2 2 0 002-2v-6a2 2 0 00-2-2H9a2 2 0 00-2 2v6a2 2 0 002 2zm7-5a2 2 0 11-4 0 2 2 0 014 0z"
            />
          </svg>
          <span>Công nợ</span>
        </button>
        <button class="sidebar-button" data-tab="supplierInfo">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
            stroke-width="2"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M17 20h2a2 2 0 002-2V7a2 2 0 00-2-2h-2v10a2 2 0 01-2 2H9m0 0a2 2 0 002 2h2a2 2 0 002-2M9 10V7a2 2 0 012-2h2a2 2 0 012 2v3m-2 0h-2"
            />
          </svg>
          <span>Thông tin NCC</span>
        </button>
        <button class="sidebar-button" data-tab="product">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
            stroke-width="2"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M10 21h7a2 2 0 002-2V9.414a1 1 0 00-.293-.707l-5.414-5.414A1 1 0 0012.586 3H7a2 2 0 00-2 2v14a2 2 0 002 2h3zm2-7l4 4m-4 0l-4-4"
            />
          </svg>
          <span>Sản phẩm</span>
        </button>
        <button class="sidebar-button" data-tab="orderHistory">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
            stroke-width="2"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M9 5H7a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2V7a2 2 0 00-2-2h-2M9 5a2 2 0 002 2h2a2 2 0 002-2M9 5a2 2 0 012-2h2a2 2 0 012 2m-3 7h3m-3 4h3m-6-4h.01M9 16h.01"
            />
          </svg>
          <span>Lịch sử đặt hàng</span>
        </button>
        <button class="sidebar-button" data-tab="settings">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
            stroke-width="2"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M10.325 4.317c.426-1.756 2.924-1.756 3.35 0a1.724 1.724 0 002.573 1.066c1.543-.94 3.31.826 2.37 2.37a1.724 1.724 0 001.065 2.572c1.756.426 1.756 2.924 0 3.35a1.724 1.724 0 00-1.066 2.573c.94 1.543-.826 3.31-2.37 2.37a1.724 1.724 0 00-2.572 1.065c-.426 1.756-2.924 1.756-3.35 0a1.724 1.724 0 00-2.573-1.066c-1.543.94-3.31-.826-2.37-2.37a1.724 1.724 0 00-1.065-2.572c-1.756-.426-1.756-2.924 0-3.35a1.724 1.724 0 001.066-2.573c-.94-1.543.826-3.31 2.37-2.37.996.608 2.296.07 2.572-1.065z"
            />
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M15 12a3 3 0 11-6 0 3 3 0 016 0z"
            />
          </svg>
          <span>Cài đặt</span>
        </button>
        <!-- Nút đăng xuất sẽ được ẩn nếu không có chức năng đăng nhập -->
        <button class="sidebar-button" onclick="window.logoutUser()">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            fill="none"
            viewBox="0 0 24 24"
            stroke-width="2"
            stroke="currentColor"
            class="w-6 h-6"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M15.75 9V5.25A2.25 2.25 0 0013.5 3h-6a2.25 2.25 0 00-2.25 2.25v13.5A2.25 2.25 0 007.5 21h6a2.25 2.25 0 002.25-2.25V15M12 9l-3 3m0 0l3 3m-3-3h12.75"
            />
          </svg>
          <span>Đăng xuất</span>
        </button>
      </nav>
    </aside>

    <!-- Main Content Area -->
    <main class="main-content">
      <!-- Dashboard Tab -->
      <div id="dashboard" class="tab-content active">
        <h2 class="text-xl font-semibold mb-4 text-gray-700">
          Dashboard - Tổng quan hiệu suất NCC
        </h2>
        <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6 mb-6">
          <div class="card">
            <h3 class="font-semibold text-lg mb-2 text-gray-700">Bộ lọc</h3>
            <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
              <div class="form-group">
                <label for="dashboard-category">Phân loại vật tư:</label>
                <select id="dashboard-category" data-type="category">
                  <option value="">Tất cả</option>
                  <!-- Options will be populated by JS -->
                </select>
              </div>
              <div class="form-group">
                <label for="dashboard-time">Thời gian:</label>
                <select id="dashboard-time">
                  <option value="last-30-days">30 ngày qua</option>
                  <option value="last-90-days">90 ngày qua</option>
                  <option value="this-year">Năm nay</option>
                  <option value="all-time">Tất cả</option>
                </select>
              </div>
              <div class="form-group col-span-full">
                <label for="dashboard-supplier">Nhà cung cấp:</label>
                <select id="dashboard-supplier" data-type="supplier">
                  <option value="">Tất cả</option>
                  <!-- Options will be populated by JS -->
                </select>
              </div>
            </div>
            <button
              class="btn btn-primary mt-4 w-full"
              onclick="window.applyDashboardFilters()"
            >
              Áp dụng bộ lọc
            </button>
          </div>
          <!-- Biểu đồ đánh giá hiệu suất của các NCC (mới) -->
          <div class="card col-span-1 md:col-span-2">
            <h3 class="font-semibold text-lg mb-2 text-gray-700">
              Biểu đồ đánh giá hiệu suất NCC (Spider Chart)
            </h3>
            <div
              class="h-64 bg-gray-100 rounded-lg flex items-center justify-center text-gray-500 relative"
            >
              <canvas id="dashboardSupplierPerformanceChart"></canvas>
              <div
                id="dashboardPerformanceChartNoDataMessage"
                class="absolute inset-0 flex items-center justify-center text-gray-500 text-center text-sm"
                style="display: none;"
              >
                Không có dữ liệu để hiển thị biểu đồ hiệu suất.
              </div>
            </div>
          </div>
        </div>

        <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
          <div class="card">
            <h3 class="font-semibold text-lg mb-2 text-gray-700">
              Điểm đánh giá NCC (có trọng số)
            </h3>
            <div id="nccScoreboard" class="overflow-x-auto">
              <table class="data-table">
                <thead>
                  <tr>
                    <th>NCC</th>
                    <th>Điểm</th>
                    <th>Tỷ lệ đúng hạn</th>
                  </tr>
                </thead>
                <tbody>
                  <!-- Data will be populated by JS -->
                </tbody>
              </table>
            </div>
          </div>
          <div class="card">
            <h3 class="font-semibold text-lg mb-2 text-gray-700">
              Tổng chi tiêu theo NCC
            </h3>
            <div id="supplierSpending" class="overflow-x-auto">
              <table class="data-table">
                <thead>
                  <tr>
                    <th>NCC</th>
                    <th>Tổng chi tiêu</th>
                  </tr>
                </thead>
                <tbody>
                  <!-- Data will be populated by JS -->
                </tbody>
              </table>
            </div>
          </div>
        </div>
        <!-- Biểu đồ tổng chi tiêu theo NCC (đã di chuyển xuống dưới) -->
        <div class="card mt-6">
          <h3 class="font-semibold text-lg mb-2 text-gray-700">
            Biểu đồ tổng chi tiêu theo NCC
          </h3>
          <div
            class="h-64 bg-gray-100 rounded-lg flex items-center justify-center text-gray-500 relative"
          >
            <canvas id="dashboardSupplierSpendingChart"></canvas>
            <div
              id="dashboardChartNoDataMessage"
              class="absolute inset-0 flex items-center justify-center text-gray-500 text-center text-sm"
              style="display: none;"
            >
              Không có dữ liệu để hiển thị biểu đồ tổng chi tiêu.
            </div>
          </div>
        </div>
      </div>

      <!-- Price Comparison Tab -->
      <div id="priceComparison" class="tab-content">
        <h2 class="text-xl font-semibold mb-4 text-gray-700">
          So sánh giá sản phẩm
        </h2>
        <div class="card mb-6">
          <h3 class="font-semibold text-lg mb-4 text-gray-700">
            Chọn sản phẩm để so sánh
          </h3>
          <div class="grid grid-cols-1 md:grid-cols-2 gap-4 items-end">
            <div class="form-group">
              <label for="priceComparisonProduct">Sản phẩm:</label>
              <select
                id="priceComparisonProduct"
                data-type="product"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white"
              ></select>
            </div>
            <div class="flex justify-end gap-4 md:col-span-1">
              <button
                class="btn btn-primary w-full md:w-auto"
                onclick="window.renderPriceComparison()"
              >
                Xem so sánh
              </button>
              <button
                class="btn btn-secondary w-full md:w-auto"
                id="updatePriceBtn"
              >
                Cập nhật giá nhanh
              </button>
            </div>
          </div>
        </div>

        <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
          <div class="card">
            <h3 class="font-semibold text-lg mb-2 text-gray-700">
              Bảng so sánh giá
            </h3>
            <div class="overflow-x-auto">
              <table class="data-table">
                <thead>
                  <tr>
                    <th>Nhà cung cấp</th>
                    <th>Giá</th>
                    <th>Ghi chú</th>
                  </tr>
                </thead>
                <tbody id="priceComparisonTableBody">
                  <!-- Data will be populated by JS -->
                </tbody>
              </table>
            </div>
          </div>
          <div class="card">
            <h3 class="font-semibold text-lg mb-2 text-gray-700">
              Biểu đồ so sánh giá
            </h3>
            <div
              class="h-64 bg-gray-100 rounded-lg flex items-center justify-center text-gray-500 relative"
            >
              <canvas id="priceComparisonChart"></canvas>
              <div
                id="priceComparisonChartNoDataMessage"
                class="absolute inset-0 flex items-center justify-center text-gray-500 text-center text-sm"
                style="display: none;"
              >
                Chọn sản phẩm để hiển thị biểu đồ.
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- Inventory Tab -->
      <div id="inventory" class="tab-content">
        <h2 class="text-xl font-semibold mb-4 text-gray-700">
          Quản lý tồn kho & Xuất/Nhập/Bàn giao
        </h2>

        <div class="card mb-6">
          <h3 class="font-semibold text-lg mb-4 text-gray-700">
            Ghi nhận Xuất/Nhập/Bàn giao
          </h3>
          <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div class="form-group">
              <label for="inventoryMovementDate">Ngày:</label>
              <input
                type="date"
                id="inventoryMovementDate"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div class="form-group">
              <label for="inventoryMovementType">Loại:</label>
              <select
                id="inventoryMovementType"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white"
              >
                <option value="Nhập">Nhập</option>
                <option value="Xuất">Xuất</option>
                <option value="Bàn giao">Bàn giao</option>
              </select>
            </div>
            <div class="form-group hidden" id="inventorySenderGroup">
              <label for="inventoryMovementSender">Người gửi:</label>
              <div class="flex items-center">
                <select
                  id="inventoryMovementSender"
                  data-type="entity-sender"
                  class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white"
                ></select>
                <button
                  type="button"
                  class="plus-button ml-2"
                  id="addInventorySenderBtn"
                >
                  +
                </button>
              </div>
            </div>
            <div class="form-group hidden" id="inventoryRecipientGroup">
              <label for="inventoryMovementRecipient">Đơn vị nhận hàng:</label>
              <div class="flex items-center">
                <select
                  id="inventoryMovementRecipient"
                  data-type="entity-recipient"
                  class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white"
                ></select>
                <button
                  type="button"
                  class="plus-button ml-2"
                  id="addInventoryRecipientBtn"
                >
                  +
                </button>
              </div>
            </div>
            <div class="form-group col-span-full">
              <label for="inventoryMovementNotes">Ghi chú:</label>
              <textarea
                id="inventoryMovementNotes"
                rows="2"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 resize-y"
              ></textarea>
            </div>
          </div>

          <h4 class="font-semibold mt-6 mb-2 text-gray-700">
            Danh sách sản phẩm:
          </h4>
          <div id="inventoryItemsContainer">
            <!-- Dynamic product items will be rendered here -->
          </div>
          <button
            type="button"
            class="btn btn-secondary mt-2 w-full"
            onclick="window.addInventoryItem()"
          >
            + Thêm sản phẩm
          </button>

          <div class="flex justify-end gap-4 mt-4">
            <button
              class="btn btn-primary"
              onclick="window.addInventoryMovement()"
            >
              Ghi nhận
            </button>
            <button
              class="btn btn-secondary"
              onclick="window.clearInventoryMovementForm()"
            >
              Làm mới
            </button>
          </div>
        </div>

        <div class="card mb-6">
          <h3 class="font-semibold text-lg mb-2 text-gray-700">
            Tổng quan tồn kho
          </h3>
          <div class="overflow-x-auto">
            <table class="data-table">
              <thead>
                <tr>
                  <th>Sản phẩm</th>
                  <th>ĐVT</th>
                  <th>Tồn kho</th>
                  <th>Ngưỡng cảnh báo</th>
                  <th>Trạng thái</th>
                </tr>
              </thead>
              <tbody id="productStockOverviewBody">
                <!-- Data will be populated by JS -->
              </tbody>
            </table>
          </div>
        </div>

        <div class="card">
          <h3 class="font-semibold text-lg mb-2 text-gray-700">
            Lịch sử xuất/nhập/bàn giao
          </h3>
          <div class="overflow-x-auto">
            <table class="data-table">
              <thead>
                <tr>
                  <th>Ngày</th>
                  <th>Loại</th>
                  <th>Sản phẩm</th>
                  <th>Người gửi</th>
                  <th>Đơn vị nhận</th>
                  <th>Ghi chú</th>
                  <th>Hành động</th>
                </tr>
              </thead>
              <tbody id="inventoryListBody">
                <!-- Data will be populated by JS -->
              </tbody>
            </table>
          </div>
        </div>
      </div>

      <!-- Debt Management Tab -->
      <div id="debtManagement" class="tab-content">
        <h2 class="text-xl font-semibold mb-4 text-gray-700">
          Quản lý Công nợ
        </h2>

        <div class="card mb-6">
          <h3
            id="debtFormTitle"
            class="font-semibold text-lg mb-4 text-gray-700"
          >
            Thêm Công Nợ Mới
          </h3>
          <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div class="form-group">
              <label for="debtDate">Ngày ghi nhận:</label>
              <input
                type="date"
                id="debtDate"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div class="form-group">
              <label for="debtSupplier">Nhà cung cấp:</label>
              <div id="debtSupplierContainer" class="flex items-center">
                <select
                  id="debtSupplier"
                  data-type="supplier"
                  class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white"
                ></select>
                <button
                  type="button"
                  class="plus-button"
                  id="addDebtSupplierBtn"
                >
                  +
                </button>
              </div>
            </div>
            <div class="form-group">
              <label for="debtAmount">Số tiền nợ:</label>
              <input
                type="number"
                id="debtAmount"
                placeholder="Số tiền"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div class="form-group">
              <label for="debtDueDate">Ngày đáo hạn:</label>
              <input
                type="date"
                id="debtDueDate"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div class="form-group">
              <label for="debtPaymentMethod">Hình thức thanh toán:</label>
              <div id="debtPaymentMethodContainer" class="flex items-center">
                <select
                  id="debtPaymentMethod"
                  data-type="payment-method"
                  class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white"
                >
                  <!-- Options populated by JS -->
                </select>
                <button type="button" class="plus-button" id="addDebtBankBtn">
                  +
                </button>
              </div>
            </div>
            <div class="form-group">
              <label for="debtInvoiceNumber">Số hóa đơn:</label>
              <input
                type="text"
                id="debtInvoiceNumber"
                placeholder="Số hóa đơn (nếu có)"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div class="form-group col-span-full">
              <label for="debtDescription">Mô tả:</label>
              <textarea
                id="debtDescription"
                rows="2"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 resize-y"
              ></textarea>
            </div>
          </div>
          <div class="flex justify-end gap-4 mt-4">
            <button
              class="btn btn-primary"
              id="addDebtBtn"
              onclick="window.addOrUpdateDebt()"
            >
              Thêm Công Nợ
            </button>
            <button
              class="btn btn-secondary hidden"
              id="cancelDebtEditBtn"
              onclick="window.clearDebtForm()"
            >
              Hủy
            </button>
          </div>
        </div>

        <div class="card mb-6">
          <h3 class="font-semibold text-lg mb-2 text-gray-700">
            Bộ lọc Công nợ
          </h3>
          <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
            <div class="form-group">
              <label for="debtFilterSupplier">Nhà cung cấp:</label>
              <select
                id="debtFilterSupplier"
                data-type="supplier"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white"
              >
                <option value="">Tất cả</option>
              </select>
            </div>
            <div class="form-group">
              <label for="debtFilterStatus">Trạng thái:</label>
              <select
                id="debtFilterStatus"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white"
              >
                <option value="">Tất cả</option>
                <option value="Chưa thanh toán">Chưa thanh toán</option>
                <option value="Đã thanh toán một phần">
                  Đã thanh toán một phần
                </option>
                <option value="Đã thanh toán">Đã thanh toán</option>
                <option value="Quá hạn">Quá hạn</option>
              </select>
            </div>
            <div class="form-group">
              <label for="debtFilterTime">Thời gian đáo hạn:</label>
              <select
                id="debtFilterTime"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white"
              >
                <option value="">Tất cả</option>
                <option value="overdue">Quá hạn</option>
                <option value="next-7-days">Trong 7 ngày tới</option>
                <option value="this-month">Trong tháng này</option>
              </select>
            </div>
          </div>
          <button
            class="btn btn-primary mt-4 w-full"
            onclick="window.applyDebtFilters()"
          >
            Áp dụng bộ lọc
          </button>
        </div>

        <div class="grid grid-cols-1 md:grid-cols-2 gap-6 mb-6">
          <div class="card">
            <h3 class="font-semibold text-lg mb-2 text-gray-700">
              Biểu đồ Công nợ theo NCC
            </h3>
            <div
              class="h-64 bg-gray-100 rounded-lg flex items-center justify-center text-gray-500 relative"
            >
              <canvas id="debtBySupplierChart"></canvas>
              <div
                id="debtBySupplierChartNoDataMessage"
                class="absolute inset-0 flex items-center justify-center text-gray-500 text-center text-sm"
                style="display: none;"
              >
                Không có dữ liệu để hiển thị biểu đồ.
              </div>
            </div>
          </div>
          <div class="card">
            <h3 class="font-semibold text-lg mb-2 text-gray-700">
              Biểu đồ Xu hướng Công nợ
            </h3>
            <div
              class="h-64 bg-gray-100 rounded-lg flex items-center justify-center text-gray-500 relative"
            >
              <canvas id="debtTrendChart"></canvas>
              <div
                id="debtTrendChartNoDataMessage"
                class="absolute inset-0 flex items-center justify-center text-gray-500 text-center text-sm"
                style="display: none;"
              >
                Không có dữ liệu để hiển thị biểu đồ.
              </div>
            </div>
          </div>
        </div>

        <div class="card">
          <h3 class="font-semibold text-lg mb-2 text-gray-700">
            Danh sách Công nợ
          </h3>
          <div class="overflow-x-auto">
            <table class="data-table">
              <thead>
                <tr>
                  <th>Ngày nợ</th>
                  <th>NCC</th>
                  <th>Mô tả</th>
                  <th>Số tiền nợ</th>
                  <th>Còn lại</th>
                  <th>Ngày đáo hạn</th>
                  <th>Trạng thái</th>
                  <th>Hành động</th>
                </tr>
              </thead>
              <tbody id="debtListBody">
                <!-- Data will be populated by JS -->
              </tbody>
            </table>
          </div>
        </div>
      </div>

      <!-- Supplier Info Tab -->
      <div id="supplierInfo" class="tab-content">
        <h2 class="text-xl font-semibold mb-4 text-gray-700">
          Thông tin Nhà cung cấp
        </h2>

        <div class="card mb-6">
          <h3
            id="supplierFormTitle"
            class="font-semibold text-lg mb-4 text-gray-700"
          >
            Thêm Nhà Cung Cấp Mới
          </h3>
          <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div class="form-group">
              <label for="supplierName">Tên Nhà cung cấp:</label>
              <input
                type="text"
                id="supplierName"
                placeholder="Tên nhà cung cấp"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div class="form-group">
              <label for="supplierCode">Mã NCC:</label>
              <input
                type="text"
                id="supplierCode"
                placeholder="Mã nhà cung cấp (tùy chọn)"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div class="form-group">
              <label for="supplierContactPerson">Người liên hệ:</label>
              <input
                type="text"
                id="supplierContactPerson"
                placeholder="Tên người liên hệ"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div class="form-group">
              <label for="supplierPhone">Điện thoại:</label>
              <input
                type="tel"
                id="supplierPhone"
                placeholder="Số điện thoại"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div class="form-group">
              <label for="supplierEmail">Email:</label>
              <input
                type="email"
                id="supplierEmail"
                placeholder="Email"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div class="form-group">
              <label for="supplierAddress">Địa chỉ:</label>
              <input
                type="text"
                id="supplierAddress"
                placeholder="Địa chỉ"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
          </div>
          <div class="flex justify-end gap-4 mt-4">
            <button
              class="btn btn-primary"
              id="addSupplierBtn"
              onclick="window.addOrUpdateSupplier()"
            >
              Thêm Nhà Cung Cấp
            </button>
            <button
              class="btn btn-secondary hidden"
              id="cancelSupplierEditBtn"
              onclick="window.clearSupplierForm()"
            >
              Hủy
            </button>
          </div>
        </div>

        <div class="card">
          <h3 class="font-semibold text-lg mb-2 text-gray-700">
            Danh sách Nhà cung cấp
          </h3>
          <div class="overflow-x-auto">
            <table class="data-table">
              <thead>
                <tr>
                  <th>Tên NCC</th>
                  <th>Mã NCC</th>
                  <th>Người liên hệ</th>
                  <th>Điện thoại</th>
                  <th>Email</th>
                  <th>Địa chỉ</th>
                  <th>Hành động</th>
                </tr>
              </thead>
              <tbody id="supplierListBody">
                <!-- Data will be populated by JS -->
              </tbody>
            </table>
          </div>
        </div>
      </div>

      <!-- Product Tab -->
      <div id="product" class="tab-content">
        <h2 class="text-xl font-semibold mb-4 text-gray-700">
          Quản lý Sản phẩm
        </h2>

        <div class="card mb-6">
          <h3
            id="productFormTitle"
            class="font-semibold text-lg mb-4 text-gray-700"
          >
            Thêm Sản Phẩm Mới
          </h3>
          <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div class="form-group">
              <label for="productName">Tên Sản phẩm:</label>
              <input
                type="text"
                id="productName"
                placeholder="Tên sản phẩm"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div class="form-group">
              <label for="productCode">Mã SP:</label>
              <input
                type="text"
                id="productCode"
                placeholder="Mã sản phẩm (tùy chọn)"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div class="form-group">
              <label for="productPrice">Giá:</label>
              <input
                type="number"
                id="productPrice"
                placeholder="Giá sản phẩm"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div class="form-group">
              <label for="productUnit">Đơn vị tính:</label>
              <div id="productUnitContainer" class="flex items-center">
                <select
                  id="productUnit"
                  data-type="unit"
                  class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white"
                ></select>
                <button
                  type="button"
                  class="plus-button"
                  id="addQuickProductUnitBtn"
                >
                  +
                </button>
              </div>
            </div>
            <div class="form-group">
              <label for="productCategory">Phân loại:</label>
              <div id="productCategoryContainer" class="flex items-center">
                <select
                  id="productCategory"
                  data-type="category"
                  class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white"
                ></select>
                <button
                  type="button"
                  class="plus-button"
                  id="addQuickProductCategoryBtn"
                >
                  +
                </button>
              </div>
            </div>
            <div class="form-group">
              <label for="productReorderPoint">Ngưỡng cảnh báo tồn kho:</label>
              <input
                type="number"
                id="productReorderPoint"
                placeholder="Số lượng cảnh báo"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div class="form-group col-span-full">
              <label for="productDescription">Mô tả:</label>
              <textarea
                id="productDescription"
                rows="3"
                placeholder="Mô tả sản phẩm"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 resize-y"
              ></textarea>
            </div>
          </div>
          <div class="flex justify-end gap-4 mt-4">
            <button
              class="btn btn-primary"
              id="addProductBtn"
              onclick="window.addOrUpdateProduct()"
            >
              Thêm Sản Phẩm
            </button>
            <button
              class="btn btn-secondary hidden"
              id="cancelProductEditBtn"
              onclick="window.clearProductForm()"
            >
              Hủy
            </button>
          </div>
        </div>

        <div class="card">
          <h3 class="font-semibold text-lg mb-2 text-gray-700">
            Danh sách Sản phẩm
          </h3>
          <div class="overflow-x-auto">
            <table class="data-table">
              <thead>
                <tr>
                  <th>Tên SP</th>
                  <th>Mã SP</th>
                  <th>Mô tả</th>
                  <th>Giá</th>
                  <th>ĐVT</th>
                  <th>Phân loại</th>
                  <th>Hành động</th>
                </tr>
              </thead>
              <tbody id="productListBody">
                <!-- Data will be populated by JS -->
              </tbody>
            </table>
          </div>
        </div>
      </div>

      <!-- Order History Tab -->
      <div id="orderHistory" class="tab-content">
        <h2 class="text-xl font-semibold mb-4 text-gray-700">
          Lịch sử đặt hàng
        </h2>

        <div class="card mb-6">
          <h3
            id="orderFormTitle"
            class="font-semibold text-lg mb-4 text-gray-700"
          >
            Thêm Đơn Hàng Mới
          </h3>
          <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div class="form-group">
              <label for="orderDate">Ngày đặt hàng:</label>
              <input
                type="date"
                id="orderDate"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div class="form-group">
              <label for="orderInvoiceNumber">Số hóa đơn:</label>
              <input
                type="text"
                id="orderInvoiceNumber"
                placeholder="Số hóa đơn (nếu có)"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div class="form-group">
              <label for="orderSupplier">Nhà cung cấp:</label>
              <div class="flex items-center">
                <select
                  id="orderSupplier"
                  data-type="supplier"
                  class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white"
                ></select>
                <button
                  type="button"
                  class="plus-button ml-2"
                  id="addOrderSupplierBtn"
                >
                  +
                </button>
              </div>
            </div>
            <div class="form-group">
              <label for="orderDeliveryDate">Ngày giao hàng dự kiến:</label>
              <input
                type="date"
                id="orderDeliveryDate"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div class="form-group">
              <label for="orderAmountPaid">Số tiền đã thanh toán:</label>
              <input
                type="number"
                id="orderAmountPaid"
                placeholder="Số tiền đã thanh toán"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div class="form-group">
              <label for="orderPaymentStatus">Trạng thái thanh toán:</label>
              <select
                id="orderPaymentStatus"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white"
              >
                <option value="Chưa thanh toán">Chưa thanh toán</option>
                <option value="Đã thanh toán một phần">
                  Đã thanh toán một phần
                </option>
                <option value="Đã thanh toán">Đã thanh toán</option>
              </select>
            </div>
            <div class="form-group col-span-full">
              <label for="orderNotes">Ghi chú:</label>
              <textarea
                id="orderNotes"
                rows="2"
                class="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 resize-y"
              ></textarea>
            </div>
          </div>

          <h4 class="font-semibold mt-6 mb-2 text-gray-700">
            Sản phẩm trong đơn hàng:
          </h4>
          <div id="orderItemsContainer">
            <!-- Dynamic product items will be rendered here -->
          </div>
          <button
            type="button"
            class="btn btn-secondary mt-2 w-full"
            onclick="window.addOrderItem()"
          >
            + Thêm sản phẩm
          </button>

          <div class="form-group mt-4">
            <label for="orderTotalAmount">Tổng số tiền:</label>
            <input
              type="text"
              id="orderTotalAmount"
              class="w-full p-3 border border-gray-300 rounded-lg bg-gray-100 cursor-not-allowed"
              readonly
              value="0 VNĐ"
            />
          </div>

          <div class="flex justify-end gap-4 mt-4">
            <button
              class="btn btn-primary"
              id="addOrderBtn"
              onclick="window.addOrUpdateOrder()"
            >
              Thêm Đơn Hàng
            </button>
            <button
              class="btn btn-secondary hidden"
              id="cancelOrderEditBtn"
              onclick="window.clearOrderForm()"
            >
              Hủy
            </button>
          </div>
        </div>

        <div class="card">
          <h3 class="font-semibold text-lg mb-2 text-gray-700">
            Danh sách Đơn hàng
          </h3>
          <div class="overflow-x-auto">
            <table class="data-table">
              <thead>
                <tr>
                  <th>Ngày đặt</th>
                  <th>Số HĐ</th>
                  <th>NCC</th>
                  <th>Sản phẩm</th>
                  <th>Tổng tiền</th>
                  <th>Đã trả</th>
                  <th>Còn lại</th>
                  <th>Trạng thái TT</th>
                  <th>Hành động</th>
                </tr>
              </thead>
              <tbody id="orderHistoryListBody">
                <!-- Data will be populated by JS -->
              </tbody>
            </table>
          </div>
        </div>
      </div>

      <!-- Settings Tab -->
      <div id="settings" class="tab-content">
        <h2 class="text-xl font-semibold mb-4 text-gray-700">
          Cài đặt ứng dụng
        </h2>

        <div class="card mb-6">
          <h3 class="font-semibold text-lg mb-4 text-gray-700">Giao diện</h3>
          <div class="flex items-center space-x-4">
            <label class="inline-flex items-center">
              <input
                type="radio"
                name="theme-option"
                value="theme-light"
                class="form-radio text-primary-color"
                onclick="window.changeTheme('theme-light')"
              />
              <span class="ml-2 text-gray-700">Sáng</span>
            </label>
            <label class="inline-flex items-center">
              <input
                type="radio"
                name="theme-option"
                value="theme-dark"
                class="form-radio text-primary-color"
                onclick="window.changeTheme('theme-dark')"
              />
              <span class="ml-2 text-gray-700">Tối</span>
            </label>
            <label class="inline-flex items-center">
              <input
                type="radio"
                name="theme-option"
                value="theme-pastel"
                class="form-radio text-primary-color"
                onclick="window.changeTheme('theme-pastel')"
              />
              <span class="ml-2 text-gray-700">Pastel</span>
            </label>
          </div>
        </div>

        <div class="card mb-6">
          <h3 class="font-semibold text-lg mb-4 text-gray-700">
            Thông tin chung
          </h3>
          <div class="flex items-center space-x-4 mb-4">
            <img
              src="https://placehold.co/100x100/6d28d9/ffffff?text=LOGO"
              alt="App Logo"
              class="w-24 h-24 rounded-full object-cover border-2 border-primary-color"
            />
            <div>
              <p class="font-medium text-lg text-gray-800">
                Tên ứng dụng: Quản Lý NCC & SP
              </p>
              <p class="text-gray-600">Phiên bản: 1.0.0</p>
              <p class="text-gray-600">
                Người dùng hiện tại:
                <span id="currentUserId"
                  >${window.userId || 'Đang tải...'}</span
                >
              </p>
            </div>
          </div>
          <button class="btn btn-secondary">Cập nhật logo</button>
        </div>

        <div class="card">
          <h3 class="font-semibold text-lg mb-4 text-gray-700">
            Quản lý dữ liệu
          </h3>
          <p class="text-gray-600 mb-4">
            Bạn có thể xuất hoặc nhập dữ liệu của mình tại đây.
          </p>
          <div class="flex gap-4">
            <button class="btn btn-primary">Xuất dữ liệu</button>
            <button class="btn btn-secondary">Nhập dữ liệu</button>
          </div>
        </div>
      </div>
    </main>
  </body>
</html>



