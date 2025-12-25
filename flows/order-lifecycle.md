# Order Lifecycle Flow

**Version:** 2.1.0
**Last Updated:** 2025-12-21
**Status:** Production (monolith + microservices hybrid)

---

## 1. Overview

Order lifecycle управляет полным циклом обработки заказов на платформе Vondi:
- От создания корзины до доставки товара
- Резервирование инвентаря и списание остатков
- Интеграция с Payment и Delivery микросервисами
- Escrow система для защиты покупателя
- State machine с валидацией переходов
- Поддержка B2C (Storefront Orders) и C2C (Marketplace Orders)

---

## 2. Order Statuses (State Machine)

### 2.1 Storefront Orders (B2C/C2C)

```
┌─────────────────────────────────────────────────────────────┐
│                   ORDER STATUS FLOW                          │
└─────────────────────────────────────────────────────────────┘

pending → confirmed → accepted → processing → shipped → delivered
   ↓          ↓          ↓            ↓          ↓
canceled  canceled  canceled     canceled  refunded

Final states: canceled, refunded
```

**Определение:** `backend/internal/domain/models/storefront_order.go`

```go
type OrderStatus string

const (
    OrderStatusPending    OrderStatus = "pending"    // Ожидает оплаты
    OrderStatusConfirmed  OrderStatus = "confirmed"  // Оплачен, подтвержден
    OrderStatusAccepted   OrderStatus = "accepted"   // Продавец принял заказ
    OrderStatusProcessing OrderStatus = "processing" // Создана отправка
    OrderStatusShipped    OrderStatus = "shipped"    // Отправлен
    OrderStatusDelivered  OrderStatus = "delivered"  // Доставлен
    OrderStatusCancelled  OrderStatus = "canceled"   // Отменен
    OrderStatusRefunded   OrderStatus = "refunded"   // Возврат средств
)
```

**Валидные переходы:**
```go
// backend/internal/proj/orders/service/order_service.go:786
validTransitions := map[OrderStatus][]OrderStatus{
    OrderStatusPending:    {OrderStatusConfirmed, OrderStatusCancelled},
    OrderStatusConfirmed:  {OrderStatusAccepted, OrderStatusCancelled, OrderStatusRefunded},
    OrderStatusAccepted:   {OrderStatusProcessing, OrderStatusCancelled},
    OrderStatusProcessing: {OrderStatusShipped, OrderStatusCancelled},
    OrderStatusShipped:    {OrderStatusDelivered, OrderStatusRefunded},
    OrderStatusDelivered:  {OrderStatusRefunded},
    OrderStatusCancelled:  {}, // Terminal
    OrderStatusRefunded:   {}, // Terminal
}
```

**Бизнес-методы:**
```go
// backend/internal/domain/models/storefront_order.go:291
func (o *StorefrontOrder) CanBeCancelled() bool {
    return o.Status == OrderStatusPending ||
           o.Status == OrderStatusConfirmed ||
           o.Status == OrderStatusAccepted
}

func (o *StorefrontOrder) CanBeAccepted() bool {
    return o.Status == OrderStatusConfirmed
}

func (o *StorefrontOrder) CanCreateShipment() bool {
    return o.Status == OrderStatusAccepted
}

func (o *StorefrontOrder) CanBeMarkedShipped() bool {
    return o.Status == OrderStatusProcessing && o.TrackingNumber != nil
}

func (o *StorefrontOrder) CanBeRefunded() bool {
    return o.Status == OrderStatusConfirmed ||
           o.Status == OrderStatusProcessing ||
           o.Status == OrderStatusShipped ||
           o.Status == OrderStatusDelivered
}
```

### 2.2 Marketplace Orders (C2C)

**Определение:** `backend/internal/domain/models/marketplace_order.go`

```go
type MarketplaceOrderStatus string

const (
    MarketplaceOrderStatusPending   = "pending"
    MarketplaceOrderStatusPaid      = "paid"
    MarketplaceOrderStatusShipped   = "shipped"
    MarketplaceOrderStatusDelivered = "delivered"
    MarketplaceOrderStatusCompleted = "completed"  // Escrow released
    MarketplaceOrderStatusDisputed  = "disputed"
    MarketplaceOrderStatusCancelled = "canceled"
    MarketplaceOrderStatusRefunded  = "refunded"
)
```

**Валидные переходы:**
```go
// marketplace_order.go:56
transitions := map[MarketplaceOrderStatus][]MarketplaceOrderStatus{
    MarketplaceOrderStatusPending:   {Paid, Cancelled},
    MarketplaceOrderStatusPaid:      {Shipped, Cancelled, Refunded},
    MarketplaceOrderStatusShipped:   {Delivered, Disputed},
    MarketplaceOrderStatusDelivered: {Completed, Disputed},
    MarketplaceOrderStatusDisputed:  {Completed, Refunded},
    // Final states
    MarketplaceOrderStatusCompleted: {},
    MarketplaceOrderStatusCancelled: {},
    MarketplaceOrderStatusRefunded:  {},
}
```

**Protection Period:**
```go
// marketplace_order.go:119
func (o *MarketplaceOrder) IsProtectionActive() bool {
    if o.ProtectionExpiresAt == nil {
        return false
    }
    return time.Now().Before(*o.ProtectionExpiresAt)
}

func (o *MarketplaceOrder) CanBeCaptured() bool {
    // Можно захватить если:
    // 1. Статус completed
    // 2. Или доставлен и истек защитный период
    if o.Status == MarketplaceOrderStatusCompleted {
        return true
    }
    if o.Status == MarketplaceOrderStatusDelivered && !o.IsProtectionActive() {
        return true
    }
    return false
}
```

### 2.3 Payment Status

**Определение:** `backend/internal/domain/models/payment_gateway.go:118`

```go
const (
    PaymentStatusPending    = "pending"
    PaymentStatusAuthorized = "authorized" // Pre-auth (hold funds)
    PaymentStatusCaptured   = "captured"   // Funds transferred
    PaymentStatusFailed     = "failed"
    PaymentStatusRefunded   = "refunded"
    PaymentStatusVoided     = "voided"
)
```

**Payment Status в контексте заказа:**
```go
// storefront_order.go:84
type StorefrontOrder struct {
    PaymentStatus        string  `json:"payment_status" db:"payment_status"`
    PaymentTransactionID *string `json:"payment_transaction_id,omitempty"`
    PaymentMethod        string  `json:"payment_method" db:"payment_method"`
}
```

**Значения:**
- `pending` - Ожидает оплаты
- `cod_pending` - COD (оплата при доставке)
- `processing` - В процессе обработки
- `completed` / `captured` - Оплачено успешно
- `failed` - Платёж не прошёл
- `refunded` - Возвращены средства
- `partially_refunded` - Частичный возврат

### 2.4 Reservation Status

**Определение:** `backend/internal/domain/models/storefront_order.go:23`

```go
type ReservationStatus string

const (
    ReservationStatusActive    = "active"    // Товар зарезервирован
    ReservationStatusCommitted = "committed" // Списан со склада
    ReservationStatusReleased  = "released"  // Освобожден
    ReservationStatusExpired   = "expired"   // Истек срок резервирования
)
```

**Inventory Reservation Entity:**
```go
// storefront_order.go:161
type InventoryReservation struct {
    ID        int64             `json:"id"`
    ProductID int64             `json:"product_id"`
    VariantID *int64            `json:"variant_id,omitempty"`
    OrderID   int64             `json:"order_id"`
    Quantity  int               `json:"quantity"`
    Status    ReservationStatus `json:"status"`
    ExpiresAt time.Time         `json:"expires_at"`
    CreatedAt time.Time         `json:"created_at"`
    UpdatedAt time.Time         `json:"updated_at"`
}

func (r *InventoryReservation) IsExpired() bool {
    return time.Now().After(r.ExpiresAt)
}
```

---

## 3. Create Order Flow

### 3.1 Shopping Cart → Order

**Endpoint:** `POST /api/v1/orders`
**Handler:** `backend/internal/proj/orders/service/order_service_tx.go:31`

```
┌──────────┐    ┌─────────────┐    ┌──────────────┐    ┌──────────┐
│ Frontend │───→│ Create Order│───→│ Reserve Stock│───→│ Calculate│
│ (Cart)   │    │  (pending)  │    │  (inventory) │    │  Totals  │
└──────────┘    └─────────────┘    └──────────────┘    └──────────┘
```

**Process:**

1. **Validate Storefront**
   ```go
   // order_service_tx.go:38
   storefront, err := s.storefrontRepo.GetByID(ctx, req.StorefrontID)
   if !storefront.IsActive {
       return fmt.Errorf("storefront is not active")
   }
   ```

2. **Get Order Items**
   ```go
   // order_service_tx.go:104
   func (s *OrderServiceTx) getOrderItems(ctx, req, userID) {
       if req.CartID != nil {
           // Из корзины
           cart, err := s.cartRepo.GetByID(ctx, *req.CartID)
           // Validate ownership
           if cart.UserID == nil || *cart.UserID != userID {
               return fmt.Errorf("cart does not belong to user")
           }
       } else {
           // Из переданных позиций (direct checkout)
           items = req.Items
       }
   }
   ```

3. **Validate Products & Stock**
   ```go
   // order_service_tx.go:188
   for _, item := range items {
       product, err := s.productRepo.GetByID(ctx, item.ProductID)
       if !product.IsActive {
           return fmt.Errorf("product %d is not active", item.ProductID)
       }

       // Check stock
       if stockQuantity < item.Quantity {
           return fmt.Errorf("insufficient stock: requested %d, available %d",
               item.Quantity, stockQuantity)
       }
   }
   ```

4. **Reserve Inventory**
   ```go
   // order_service_tx.go:224
   reservation, err := s.reserveStockInTx(ctx, tx,
       item.ProductID, item.VariantID, item.Quantity, order.ID)

   // Note: In production this calls inventoryMgr.ReserveStock which:
   // - Creates InventoryReservation with status: active
   // - Sets ExpiresAt: now + 15 minutes
   // - Updates product reserved_quantity
   ```

5. **Create Order (status: pending)**
   ```go
   // order_service_tx.go:145
   order := &StorefrontOrder{
       OrderNumber:     fmt.Sprintf("ORD-%d-%d-%d", storefrontID, userID, timestamp),
       Status:          OrderStatusPending,
       PaymentStatus:   "pending",
       EscrowDays:      calculateEscrowDays(storefront),
       ShippingAddress: convertToJSONB(req.ShippingAddress),
       BillingAddress:  convertToJSONB(req.BillingAddress),
       PaymentMethod:   req.PaymentMethod,
       Currency:        "RSD", // Default
       // Initialize financial fields
       SubtotalAmount:   decimal.Zero,
       TaxAmount:        decimal.Zero,
       ShippingAmount:   decimal.Zero,
       TotalAmount:      decimal.Zero,
   }
   ```

6. **Calculate Totals**
   ```go
   // order_service.go:322
   func calculateOrderTotals(order, storefront) {
       // Subtotal (sum of items)
       order.SubtotalAmount = sum(item.TotalPrice)

       // Shipping (via Delivery Service gRPC)
       order.ShippingAmount = calculateShippingCost(order, storefront)

       // Tax (VAT for Serbia: 20%, льготная: 10%)
       order.TaxAmount = order.SubtotalAmount * 0.20

       // Total
       order.TotalAmount = Subtotal + Shipping + Tax

       // Platform commission (depends on subscription plan)
       commissionRate := map[SubscriptionPlan]float64{
           SubscriptionPlanStarter:      0.03,  // 3%
           SubscriptionPlanProfessional: 0.02,  // 2%
           SubscriptionPlanBusiness:     0.01,  // 1%
           SubscriptionPlanEnterprise:   0.005, // 0.5%
       }[storefront.SubscriptionPlan]

       order.CommissionAmount = TotalAmount * commissionRate
       order.SellerAmount = TotalAmount - CommissionAmount

       // Update aliases for backward compatibility
       order.Subtotal = order.SubtotalAmount
       order.Tax = order.TaxAmount
       order.Shipping = order.ShippingAmount
       order.Total = order.TotalAmount
   }
   ```

7. **Clear Cart** (if created from cart)
   ```go
   // order_service_tx.go:87
   if req.CartID != nil {
       s.clearCartInTx(ctx, tx, *req.CartID)
   }
   ```

**Response:**
```json
{
  "id": 123,
  "order_number": "ORD-1-42-1703174400",
  "status": "pending",
  "payment_status": "pending",
  "total_amount": "1500.00",
  "currency": "RSD",
  "escrow_days": 7,
  "items": [
    {
      "product_id": 456,
      "product_name": "Product Name",
      "quantity": 2,
      "price_per_unit": "500.00",
      "total_price": "1000.00"
    }
  ]
}
```

---

## 4. Payment Flow

### 4.1 Create Payment Session

**Endpoint:** `POST /api/v1/orders/{id}/payment`
**Handler:** `backend/internal/proj/payments/handler/order_payment_handler.go:89`

**Request:**
```json
{
  "order_id": 123,
  "amount": "1500.00",
  "currency": "RSD",
  "payment_method": "card",
  "provider": "allsecure",
  "return_url": "https://vondi.rs/orders/123/success",
  "cancel_url": "https://vondi.rs/orders/123/cancel",
  "customer_email": "buyer@example.com",
  "escrow_enabled": true
}
```

**Process:**

1. **Get Order** (via Listings gRPC)
   ```go
   // order_payment_handler.go:143
   userID, _ := authMiddleware.GetUserID(c)
   userUUID, _ := authMiddleware.GetUserUUID(c)

   getOrderResp, err := h.listingsClient.GetOrder(ctx, &orderspb.GetOrderRequest{
       OrderId: int64(orderID),
       UserId:  &userID64,
   })
   order := getOrderResp.Order
   ```

2. **Validate Ownership**
   ```go
   // order_payment_handler.go:161
   if order.UserId != nil && *order.UserId != int64(userID) {
       h.logger.Warn("User %d attempted to pay for order %d belonging to user %d", ...)
       return utils.ErrorResponse(c, 403, "access denied")
   }
   ```

3. **Get Allowed Payment Methods**
   ```go
   // order_payment_handler.go:167
   paymentMethodsResp, err := h.listingsClient.GetPaymentMethods(ctx,
       &orderspb.GetPaymentMethodsRequest{
           StorefrontId: order.StorefrontId,
       })

   // Validate selected method is allowed and enabled
   methodAllowed := false
   for _, method := range paymentMethodsResp.Methods {
       if matchesPaymentMethod(method, req.PaymentMethod) && method.IsEnabled {
           methodAllowed = true
           break
       }
   }
   ```

4. **Route Payment** (based on method)

   **Online Payment (card):**
   ```go
   // order_payment_handler.go:200+
   // Call Payment Service gRPC
   createResp, err := h.paymentClient.CreateCheckoutSession(ctx, &pb.CreateCheckoutSessionRequest{
       UserUuid:      userUUID,
       Amount:        req.Amount,
       Currency:      req.Currency,
       PaymentMethod: pb.PaymentMethod_PAYMENT_METHOD_CARD,
       Provider:      pb.Provider_PROVIDER_ALLSECURE,
       SuccessUrl:    req.ReturnURL,
       CancelUrl:     req.CancelURL,
       Metadata: map[string]string{
           "order_id":       strconv.Itoa(orderID),
           "storefront_id":  strconv.Itoa(int(order.StorefrontId)),
       },
   })

   // Response: payment_url (redirect to payment gateway)
   return CreateOrderPaymentResponse{
       SessionID:      createResp.SessionId,
       PaymentURL:     createResp.PaymentUrl,
       Status:         "pending",
       RequiresAction: true,
       ExpiresAt:      time.Now().Add(30 * time.Minute).Format(time.RFC3339),
   }
   ```

   **Offline Payment (cod, cash, bank_transfer):**
   ```go
   // order_payment_handler.go:200+ (offline path)
   // No external provider needed
   // Update order status directly via Listings Service
   updateResp, err := h.listingsClient.UpdateOrderPaymentStatus(ctx,
       &orderspb.UpdateOrderPaymentStatusRequest{
           OrderId:       int64(orderID),
           PaymentStatus: "cod_pending",
           PaymentMethod: req.PaymentMethod,
       })

   // Auto-confirm COD orders
   if req.PaymentMethod == "cod" || req.PaymentMethod == "cash_on_delivery" {
       _, err = h.listingsClient.ConfirmOrder(ctx, &orderspb.ConfirmOrderRequest{
           OrderId: int64(orderID),
       })
   }

   return CreateOrderPaymentResponse{
       SessionID:      fmt.Sprintf("offline_%d_%d", orderID, time.Now().Unix()),
       Status:         "pending_manual",
       RequiresAction: false, // No payment gateway redirect needed
   }
   ```

**Response:**
```json
{
  "session_id": "ps_abc123",
  "payment_url": "https://secure.allsecure.com/...",
  "status": "pending",
  "requires_action": true,
  "expires_at": "2025-12-21T15:30:00Z"
}
```

### 4.2 Payment Callback (Webhook)

**Endpoint:** `POST /api/v1/payments/webhook/{provider}`
**Handler:** `backend/internal/proj/payments/handler/webhook_payment_service_handler.go`

**Process:**

1. **Verify Webhook Signature** (HMAC, AllSecure, etc.)

2. **Parse Event**
   ```json
   {
     "type": "payment.succeeded",
     "session_id": "ps_abc123",
     "transaction_id": "txn_xyz789",
     "order_id": "123",
     "amount": "1500.00",
     "currency": "RSD",
     "status": "captured"
   }
   ```

3. **Update Order Status**
   ```go
   // backend/internal/proj/orders/service/order_service.go:79
   func (s *OrderService) ConfirmOrder(ctx context.Context, orderID int64) error {
       s.logger.Info("Confirming order (order_id: %d)", orderID)

       order, err := s.orderRepo.GetByID(ctx, orderID)
       if order.Status != models.OrderStatusPending {
           return fmt.Errorf("order is not in pending status: %s", order.Status)
       }

       // 1. Commit reservations (списать товар со склада)
       if err := s.inventoryMgr.CommitOrderReservations(ctx, orderID); err != nil {
           return fmt.Errorf("failed to commit reservations: %w", err)
       }

       // 2. Update order status
       now := time.Now()
       order.Status = models.OrderStatusConfirmed
       order.PaymentStatus = "captured"
       order.ConfirmedAt = &now

       if err := s.orderRepo.Update(ctx, order); err != nil {
           return fmt.Errorf("failed to update order status: %w", err)
       }

       // 3. Create shipment (if delivery method selected)
       if err := s.createShipmentForOrder(ctx, order); err != nil {
           s.logger.Error("Failed to create shipment: %v (order_id: %d)", err, orderID)
           // НЕ фейлим весь order - shipment можно создать позже вручную
       }

       s.logger.Info("Order confirmed successfully (order_id: %d)", orderID)
       return nil
   }
   ```

4. **Create Shipment** (via Delivery Service gRPC)
   ```go
   // order_service.go:824
   func (s *OrderService) createShipmentForOrder(ctx, order) error {
       // Validate addresses
       if len(order.ShippingAddress) == 0 {
           return fmt.Errorf("missing shipping address")
       }

       toStreet, _ := order.ShippingAddress["street"].(string)
       toCity, _ := order.ShippingAddress["city"].(string)
       toPostalCode, _ := order.ShippingAddress["postal_code"].(string)

       // Get storefront for sender address
       storefront, err := s.storefrontRepo.GetByID(ctx, order.StorefrontID)

       // Map provider string → pb.DeliveryProvider
       provider := *order.ShippingProvider
       pbProvider := mapProviderToPb(provider) // post_express, bex, aks, etc.

       // Call Delivery Service
       shipmentResp, err := s.deliveryClient.CreateShipment(ctx, &deliveryv1.CreateShipmentRequest{
           Provider: pbProvider,
           FromAddress: &deliveryv1.Address{
               Street:     *storefront.Address,
               City:       *storefront.City,
               PostalCode: *storefront.PostalCode,
               Country:    "RS", // Serbia
           },
           ToAddress: &deliveryv1.Address{
               Street:     toStreet,
               City:       toCity,
               PostalCode: toPostalCode,
               Country:    "RS",
           },
           Package: &deliveryv1.Package{
               Weight:        strconv.FormatFloat(totalWeight, 'f', 2, 32),
               Description:   fmt.Sprintf("Order #%s", order.OrderNumber),
               DeclaredValue: order.TotalAmount.StringFixed(2),
           },
       })

       // Update order with tracking info
       trackingNumber := shipmentResp.Shipment.TrackingNumber
       order.TrackingNumber = &trackingNumber
       order.DeliveryProvider = &provider
       order.Status = models.OrderStatusProcessing

       if err := s.orderRepo.Update(ctx, order); err != nil {
           // ВАЖНО: Shipment УЖЕ создан в microservice!
           s.logger.Error("Shipment created but failed to update order: %v", err)
           return fmt.Errorf("failed to update order with shipment: %w", err)
       }

       s.logger.Info("Shipment created (order_id: %d, tracking: %s)",
           order.ID, trackingNumber)
       return nil
   }
   ```

---

## 5. Fulfillment Flow (Seller Actions)

### 5.1 Accept Order

**Endpoint:** `PUT /api/v1/storefronts/{storefront_id}/orders/{order_id}/status`
**Handler:** `backend/internal/proj/orders/service/order_service.go:188`

**Actor:** Seller
**Transition:** `confirmed → accepted`

```go
func (s *OrderService) UpdateOrderStatus(ctx, orderID, storefrontID, userID, status, notes) {
    // Validate ownership
    storefront, err := s.storefrontRepo.GetByID(ctx, order.StorefrontID)
    if storefront.UserID != userID {
        return fmt.Errorf("access denied")
    }

    // Validate transition
    if !isValidStatusTransition(order.Status, status) {
        return fmt.Errorf("invalid status transition: %s → %s",
            order.Status, status)
    }

    // Update
    order.Status = OrderStatusAccepted
    order.AcceptedAt = &now
    if notes != nil {
        order.SellerNotes = notes
    }

    s.orderRepo.Update(ctx, order)
}
```

### 5.2 Create Shipment

**Actor:** Seller (or auto after payment confirmation)
**Transition:** `accepted → processing`

```go
// Called automatically in ConfirmOrder or manually by seller
func (s *OrderService) createShipmentForOrder(ctx, order) error {
    // See section 4.2 (Payment Callback) for full implementation
}
```

### 5.3 Mark as Shipped

**Actor:** Seller (or auto from delivery webhook)
**Transition:** `processing → shipped`

```go
order.Status = OrderStatusShipped
order.ShippedAt = &now

// TrackingNumber already set when shipment was created
s.logger.Info("Order shipped (order_id: %d, tracking: %s)",
    order.ID, *order.TrackingNumber)
```

---

## 6. Delivery Flow

### 6.1 Track Shipment

**Endpoint:** `GET /api/v1/orders/{id}`
**Enrichment:** Real-time tracking data

```go
// backend/internal/proj/orders/service/order_service.go:930
func (s *OrderService) enrichOrderWithTracking(ctx, order) error {
    // Check if tracking available
    if s.deliveryClient == nil || order.TrackingNumber == nil {
        return nil // Graceful degradation
    }

    // Call Delivery Service
    trackResp, err := s.deliveryClient.TrackShipment(ctx, &deliveryv1.TrackShipmentRequest{
        TrackingNumber: *order.TrackingNumber,
    })
    if err != nil {
        s.logger.Warn("Failed to fetch tracking: %v (order: %d)", err, order.ID)
        return nil // Don't fail entire GetOrder
    }

    // Enrich order (NOT stored in DB, only in response)
    if trackResp.Shipment != nil {
        statusStr := trackResp.Shipment.Status.String()
        order.DeliveryStatus = &statusStr

        if trackResp.Shipment.EstimatedDelivery != nil {
            estDelivery := trackResp.Shipment.EstimatedDelivery.AsTime().Format(time.RFC3339)
            order.EstimatedDelivery = &estDelivery
        }

        if trackResp.Shipment.ActualDelivery != nil {
            actualDelivery := trackResp.Shipment.ActualDelivery.AsTime().Format(time.RFC3339)
            order.ActualDelivery = &actualDelivery
        }
    }

    return nil
}
```

**Response:**
```json
{
  "id": 123,
  "order_number": "ORD-1-42-1703174400",
  "status": "shipped",
  "payment_status": "captured",
  "tracking_number": "PN123456789",
  "delivery_provider": "post_express",

  // Enriched fields (not in DB)
  "delivery_status": "in_transit",
  "estimated_delivery": "2025-12-25T12:00:00Z",
  "tracking_events": [
    {
      "timestamp": "2025-12-21T10:00:00Z",
      "status": "picked_up",
      "location": "Belgrade Warehouse"
    },
    {
      "timestamp": "2025-12-21T14:00:00Z",
      "status": "in_transit",
      "location": "Novi Sad Hub"
    }
  ]
}
```

### 6.2 Delivery Webhook

**Endpoint:** `POST /api/v1/delivery/webhook/{provider}`

**Event Types:**
- `shipment.picked_up`
- `shipment.in_transit`
- `shipment.out_for_delivery`
- `shipment.delivered`
- `shipment.failed`

**Process:**
```go
// On delivery confirmation
order.Status = OrderStatusDelivered
order.DeliveredAt = &now

// Calculate escrow release date
expiresAt := now.Add(time.Duration(order.EscrowDays) * 24 * time.Hour)
order.EscrowReleaseDate = &expiresAt

s.logger.Info("Order delivered (order_id: %d, escrow_expires: %s)",
    order.ID, expiresAt.Format(time.RFC3339))
```

---

## 7. Completion & Escrow Flow

### 7.1 Escrow System

**Purpose:** Защита покупателя (buyer protection period)

**Escrow Duration** (depends on subscription plan):
```go
// backend/internal/proj/orders/service/order_service.go:308
func calculateEscrowDays(storefront *Storefront) int {
    switch storefront.SubscriptionPlan {
    case SubscriptionPlanBusiness, SubscriptionPlanEnterprise:
        return 3 // Premium sellers (trusted)
    case SubscriptionPlanProfessional:
        return 5 // Medium tier
    case SubscriptionPlanStarter:
        return 7 // Default / new sellers
    default:
        return 7 // Fallback
    }
}
```

**Marketplace Order Protection:**
```go
// marketplace_order.go:120
func (o *MarketplaceOrder) SetProtectionExpiry() {
    if o.DeliveredAt != nil {
        expiresAt := o.DeliveredAt.Add(
            time.Duration(o.ProtectionPeriodDays) * 24 * time.Hour)
        o.ProtectionExpiresAt = &expiresAt
    }
}

func (o *MarketplaceOrder) IsProtectionActive() bool {
    if o.ProtectionExpiresAt == nil {
        return false
    }
    return time.Now().Before(*o.ProtectionExpiresAt)
}

func (o *MarketplaceOrder) CanBeCaptured() bool {
    // Можно захватить платеж если:
    if o.Status == MarketplaceOrderStatusCompleted {
        return true // Buyer confirmed
    }
    if o.Status == MarketplaceOrderStatusDelivered && !o.IsProtectionActive() {
        return true // Protection expired
    }
    return false
}
```

**Auto-completion (Cron Job):**
```go
// Pseudo-code for scheduled job
func AutoCompleteOrders() {
    orders := getOrdersWithExpiredEscrow()

    for _, order := range orders {
        if order.Status == OrderStatusDelivered &&
           time.Now().After(*order.EscrowReleaseDate) {

            // Auto-complete order
            order.Status = OrderStatusCompleted

            // Capture payment (transfer to seller)
            paymentService.CapturePayment(order.PaymentTransactionID)

            logger.Info("Order auto-completed after escrow (order_id: %d)", order.ID)
        }
    }
}
```

### 7.2 Manual Completion

**Endpoint:** `POST /api/v1/orders/{id}/complete`
**Actor:** Buyer

```go
// Buyer confirms receipt before escrow expires
order.Status = OrderStatusCompleted

// Release escrow immediately
paymentService.CapturePayment(order.PaymentTransactionID)

logger.Info("Order manually completed by buyer (order_id: %d)", order.ID)
```

---

## 8. Cancellation Flow

**Endpoint:** `POST /api/v1/orders/{id}/cancel`
**Handler:** `backend/internal/proj/orders/service/order_service.go:117`

**Allowed Statuses:**
```go
func (o *StorefrontOrder) CanBeCancelled() bool {
    return o.Status == OrderStatusPending ||
           o.Status == OrderStatusConfirmed ||
           o.Status == OrderStatusAccepted
}
```

**Process:**
```go
func (s *OrderService) CancelOrder(ctx, orderID, userID, reason) (*StorefrontOrder, error) {
    order, err := s.orderRepo.GetByID(ctx, orderID)

    // Validate ownership
    if order.CustomerID != userID {
        return nil, fmt.Errorf("access denied")
    }

    // Validate status
    if !order.CanBeCancelled() {
        return nil, fmt.Errorf("cannot cancel order in status: %s", order.Status)
    }

    // 1. Release inventory reservations
    if err := s.inventoryMgr.ReleaseOrderReservations(ctx, orderID); err != nil {
        s.logger.Error("Failed to release reservations: %v (order: %d)", err, orderID)
    }

    // 2. Void payment (if authorized)
    if order.PaymentTransactionID != nil && order.PaymentStatus == "authorized" {
        paymentService.VoidPayment(*order.PaymentTransactionID)
    }

    // 3. Cancel shipment (if created)
    if order.ShipmentID != nil {
        deliveryClient.CancelShipment(ctx, *order.ShipmentID)
    }

    // 4. Update order
    now := time.Now()
    order.Status = OrderStatusCancelled
    order.PaymentStatus = "voided"
    order.CancelledAt = &now
    if reason != "" {
        order.CustomerNotes = &reason
    }

    s.orderRepo.Update(ctx, order)

    s.logger.Info("Order cancelled (order_id: %d, reason: %s)", orderID, reason)
    return order, nil
}
```

---

## 9. Refund Flow

**Endpoint:** `POST /api/v1/orders/{id}/refund`

**Allowed Statuses:**
```go
func (o *StorefrontOrder) CanBeRefunded() bool {
    return o.Status == OrderStatusConfirmed ||
           o.Status == OrderStatusProcessing ||
           o.Status == OrderStatusShipped ||
           o.Status == OrderStatusDelivered
}
```

**Process:**
```go
// 1. Create refund request
refundReq := &RefundRequest{
    OrderID: orderID,
    Amount:  order.TotalAmount,
    Reason:  "Customer not satisfied",
}

// 2. Call Payment Service
refundResp, err := paymentClient.CreateRefund(ctx, &pb.CreateRefundRequest{
    TransactionId: order.PaymentTransactionID,
    Amount:        order.TotalAmount.String(),
    Reason:        refundReq.Reason,
})

// 3. Update order
order.Status = OrderStatusRefunded
order.PaymentStatus = "refunded"

// 4. Restore inventory (if not shipped yet)
if order.Status == OrderStatusConfirmed || order.Status == OrderStatusAccepted {
    inventoryMgr.RestoreStock(ctx, orderID)
}

logger.Info("Order refunded (order_id: %d, amount: %s)",
    order.ID, order.TotalAmount.String())
```

---

## 10. COD (Cash on Delivery) Flow

### 10.1 COD Order Creation

**Special Handling:**
```go
// Auto-confirm COD orders (no payment gateway needed)
if req.PaymentMethod == "cod" ||
   req.PaymentMethod == "cash_on_delivery" ||
   req.PaymentMethod == "pouzecem" {

    order.Status = OrderStatusConfirmed // Skip 'pending'
    order.PaymentStatus = "cod_pending" // NOT 'completed'
    order.ConfirmedAt = &now

    // Reserve inventory immediately
    s.inventoryMgr.CommitOrderReservations(ctx, order.ID)
}
```

### 10.2 COD Shipment Creation

**Can create shipment from 'confirmed' status:**
```go
func (o *Order) CanCreateShipment() bool {
    if o.Status == OrderStatusAccepted {
        return true // Normal flow
    }

    // COD orders can create shipment from 'confirmed'
    if o.Status == OrderStatusConfirmed && o.IsCOD() {
        return true
    }

    return false
}

func (o *Order) IsCOD() bool {
    method := *o.PaymentMethod
    return method == "cod" ||
           method == "cash_on_delivery" ||
           method == "cash" ||
           method == "pouzecem" ||
           method == "pouzećem"
}
```

### 10.3 COD Payment Finalization

**After delivery:**
```go
// Delivery webhook callback
if order.IsCOD() && deliveryStatus == "delivered" {
    // Courier collected payment
    order.Status = OrderStatusDelivered
    order.PaymentStatus = "completed" // NOW payment is complete
    order.DeliveredAt = &now

    // Set escrow (seller gets funds after escrow expires)
    order.EscrowReleaseDate = &now.Add(time.Duration(order.EscrowDays) * 24 * time.Hour)
}
```

---

## 11. Sequence Diagrams

### 11.1 Complete Order Flow (Online Payment)

```
┌─────────┐  ┌──────────┐  ┌────────┐  ┌─────────┐  ┌──────────┐  ┌──────────┐
│ Buyer   │  │ Frontend │  │ Backend│  │ Payment │  │ Delivery │  │ Seller   │
└────┬────┘  └────┬─────┘  └───┬────┘  └────┬────┘  └────┬─────┘  └────┬─────┘
     │            │             │            │            │             │
     │ Add to Cart│             │            │            │             │
     ├───────────►│             │            │            │             │
     │            │ POST /cart  │            │            │             │
     │            ├────────────►│            │            │             │
     │            │             │ Reserve    │            │             │
     │            │             │ Stock      │            │             │
     │            │◄────────────┤            │            │             │
     │◄───────────┤             │            │            │             │
     │            │             │            │            │             │
     │ Checkout   │             │            │            │             │
     ├───────────►│ POST /orders│            │            │             │
     │            ├────────────►│            │            │             │
     │            │             │ Create Order            │             │
     │            │             │ (pending)  │            │             │
     │            │◄────────────┤            │            │             │
     │◄───────────┤             │            │            │             │
     │            │             │            │            │             │
     │ Pay        │             │            │            │             │
     ├───────────►│ POST /payment            │            │             │
     │            ├────────────►│            │            │             │
     │            │             ├───────────►│            │             │
     │            │             │ Create     │            │             │
     │            │             │ Session    │            │             │
     │            │◄────────────┤            │            │             │
     │◄───────────┤ payment_url │            │             │             │
     │            │             │            │            │             │
     │ Complete Payment         │            │            │             │
     ├──────────────────────────┼───────────►│            │             │
     │            │             │            │            │             │
     │            │             │◄───────────┤ Webhook    │             │
     │            │             │ (success)  │            │             │
     │            │             │            │            │             │
     │            │             │ Confirm Order           │             │
     │            │             │ Commit Stock            │             │
     │            │             │ Create Shipment         │             │
     │            │             ├────────────┼───────────►│             │
     │            │             │            │            │             │
     │            │             │            │ Tracking # │             │
     │            │             │◄───────────┼────────────┤             │
     │            │             │            │            │             │
     │            │             │            │            │ Notification│
     │            │             │            │            │ (new order) │
     │            │             ├────────────┼────────────┼────────────►│
     │            │             │            │            │             │
     │            │             │            │            │             │ Accept
     │            │             │◄───────────┼────────────┼─────────────┤
     │            │             │            │            │             │
     │            │             │            │            │             │ Ship
     │            │             │◄───────────┼────────────┼─────────────┤
     │            │             │            │            │             │
     │            │             │            │ Webhook    │             │
     │            │             │◄───────────┼────────────┤             │
     │            │             │ (delivered)│            │             │
     │            │             │            │            │             │
     │            │             │ Set Escrow │            │             │
     │            │             │ Expiry     │            │             │
     │            │             │            │            │             │
     │            │             │            │            │             │
     │   (7 days later - escrow expires)     │            │             │
     │            │             │            │            │             │
     │            │             │ Auto-Complete           │             │
     │            │             │ Capture Payment         │             │
     │            │             ├───────────►│            │             │
     │            │             │            │            │             │
     │            │             │            │ Transfer   │             │
     │            │             │            │ to Seller  │             │
     │            │             │            ├────────────┼────────────►│
     └────────────┴─────────────┴────────────┴────────────┴─────────────┘
```

---

## 12. API Endpoints

### 12.1 Order Management

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| POST | `/api/v1/orders` | Create order | User |
| GET | `/api/v1/orders/{id}` | Get order details | User |
| GET | `/api/v1/orders` | List user orders | User |
| POST | `/api/v1/orders/{id}/cancel` | Cancel order | User |
| POST | `/api/v1/orders/{id}/complete` | Complete order (buyer) | User |

### 12.2 Seller Order Management

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| GET | `/api/v1/storefronts/{id}/orders` | List storefront orders | Seller |
| PUT | `/api/v1/storefronts/{id}/orders/{order_id}/status` | Update order status | Seller |
| POST | `/api/v1/storefronts/{id}/orders/{order_id}/accept` | Accept order | Seller |
| POST | `/api/v1/storefronts/{id}/orders/{order_id}/ship` | Mark as shipped | Seller |

### 12.3 Payment

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| POST | `/api/v1/orders/{id}/payment` | Create payment session | User |
| POST | `/api/v1/payments/webhook/{provider}` | Payment webhook | Public (verified) |

### 12.4 Cart

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| POST | `/api/v1/carts/items` | Add item to cart | User/Session |
| GET | `/api/v1/carts` | Get cart | User/Session |
| PUT | `/api/v1/carts/items/{id}` | Update cart item | User/Session |
| DELETE | `/api/v1/carts/items/{id}` | Remove from cart | User/Session |
| DELETE | `/api/v1/carts` | Clear cart | User/Session |

---

## 13. Database Schema

### 13.1 orders (vondi_db)

**Location:** `vondi_db.orders`
**Migration:** `backend/migrations/000082_orders.up.sql`

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    order_number VARCHAR(50) UNIQUE NOT NULL,  -- ORD-{sf_id}-{user_id}-{timestamp}
    storefront_id INTEGER NOT NULL REFERENCES storefronts(id),
    customer_id INTEGER NOT NULL REFERENCES users(id),

    -- Financial
    subtotal DECIMAL(12,2) NOT NULL,
    tax DECIMAL(12,2) DEFAULT 0,
    shipping DECIMAL(12,2) DEFAULT 0,
    discount DECIMAL(12,2) DEFAULT 0,
    total DECIMAL(12,2) NOT NULL,
    commission_amount DECIMAL(12,2) NOT NULL,
    seller_amount DECIMAL(12,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'RSD',

    -- Status
    status order_status NOT NULL DEFAULT 'pending',
    payment_status VARCHAR(50) DEFAULT 'pending',
    payment_transaction_id VARCHAR(255),
    payment_method VARCHAR(50),

    -- Escrow
    escrow_days INTEGER DEFAULT 7,
    escrow_release_date TIMESTAMP,

    -- Delivery
    shipping_method VARCHAR(100),
    shipping_provider VARCHAR(100),
    tracking_number VARCHAR(255),
    shipment_id BIGINT,
    delivery_provider VARCHAR(50),
    label_url TEXT,

    -- Addresses (JSONB)
    shipping_address JSONB NOT NULL,
    billing_address JSONB,
    pickup_address JSONB,

    -- Metadata
    notes TEXT,
    customer_notes TEXT,
    seller_notes TEXT,
    metadata JSONB,

    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    confirmed_at TIMESTAMP,
    accepted_at TIMESTAMP,
    shipped_at TIMESTAMP,
    delivered_at TIMESTAMP,
    canceled_at TIMESTAMP,

    -- Indexes
    INDEX idx_orders_customer_id (customer_id),
    INDEX idx_orders_storefront_id (storefront_id),
    INDEX idx_orders_status (status),
    INDEX idx_orders_order_number (order_number),
    INDEX idx_orders_tracking_number (tracking_number)
);
```

### 13.2 order_items

```sql
CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL,
    variant_id BIGINT,

    -- Snapshot data (immutable)
    product_name VARCHAR(500) NOT NULL,
    product_sku VARCHAR(100),
    product_image_url TEXT,
    variant_name VARCHAR(255),

    quantity INTEGER NOT NULL CHECK (quantity > 0),
    price_per_unit DECIMAL(12,2) NOT NULL,
    total_price DECIMAL(12,2) NOT NULL,

    product_attributes JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_order_items_order_id (order_id)
);
```

### 13.3 inventory_reservations

```sql
CREATE TABLE inventory_reservations (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL,
    variant_id BIGINT,
    order_id BIGINT NOT NULL REFERENCES orders(id),

    quantity INTEGER NOT NULL,
    status reservation_status NOT NULL DEFAULT 'active',

    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_reservations_product_id (product_id),
    INDEX idx_reservations_order_id (order_id),
    INDEX idx_reservations_status (status),
    INDEX idx_reservations_expires_at (expires_at)
);
```

### 13.4 shopping_carts

```sql
CREATE TABLE shopping_carts (
    id BIGSERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    storefront_id INTEGER NOT NULL REFERENCES storefronts(id),
    session_id VARCHAR(255),

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE (user_id, storefront_id),
    UNIQUE (session_id, storefront_id)
);
```

### 13.5 shopping_cart_items

```sql
CREATE TABLE shopping_cart_items (
    id BIGSERIAL PRIMARY KEY,
    cart_id BIGINT NOT NULL REFERENCES shopping_carts(id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL,
    variant_id BIGINT,

    quantity INTEGER NOT NULL CHECK (quantity > 0),
    price_per_unit DECIMAL(12,2) NOT NULL,
    total_price DECIMAL(12,2) NOT NULL,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE (cart_id, product_id, variant_id),
    INDEX idx_cart_items_cart_id (cart_id)
);
```

---

## 14. Integration Points

### 14.1 Microservices

| Service | Purpose | Protocol | Port |
|---------|---------|----------|------|
| **Listings** | Order CRUD, Cart, Inventory | gRPC | 50053 |
| **Payment** | Payment sessions, Capture, Refund | gRPC | 50051 |
| **Delivery** | Shipment creation, Tracking | gRPC | 50052 |
| **Auth** | User authentication, JWT validation | HTTP/gRPC | 28086 |

**gRPC Clients:**
```go
// backend/internal/clients/listings/client.go
listingsClient := listings.NewClient("localhost:50053", logger)

// backend/internal/clients/payment/client.go
paymentClient := payment.NewClient("localhost:50051", logger)

// backend/internal/proj/delivery/grpcclient/client.go
deliveryClient := grpcclient.NewClient("localhost:50052", logger)
```

### 14.2 External APIs

| Provider | Purpose | Integration | Credentials |
|----------|---------|-------------|-------------|
| **AllSecure** | Online payments (card) | Webhook + API | API Key |
| **Post Express** | Delivery (Serbia) | API + Webhook | API Key |
| **MinIO** | File storage (labels, receipts) | S3 API | Access Key |

---

## 15. Error Handling

### 15.1 Common Errors

| Error | HTTP Status | Reason | Resolution |
|-------|-------------|--------|------------|
| `insufficient_stock` | 400 | Not enough inventory | Wait for restock or reduce quantity |
| `invalid_payment_method` | 400 | Method not enabled | Choose allowed method |
| `order_not_found` | 404 | Invalid order ID | Check order ID |
| `invalid_status_transition` | 400 | Cannot change status | Follow valid transitions |
| `payment_failed` | 402 | Payment declined | Use different card or method |
| `escrow_active` | 409 | Cannot refund during protection | Wait for escrow expiry |
| `access_denied` | 403 | User not authorized | Check ownership |
| `cart_empty` | 400 | No items in cart | Add items first |

### 15.2 Graceful Degradation

**Delivery Service Unavailable:**
```go
// order_service.go:654
if s.deliveryClient == nil {
    s.logger.Warn("Delivery client not available, using fallback rate")
    return decimal.NewFromFloat(100.0) // Graceful degradation
}
```

**Payment Service Unavailable:**
```go
// Allow offline payment methods
if req.PaymentMethod == "cod" || req.PaymentMethod == "cash" {
    order.PaymentStatus = "pending_manual"
    return nil // No payment gateway needed
}
```

**Tracking Info Unavailable:**
```go
// order_service.go:930
func enrichOrderWithTracking(ctx, order) error {
    trackResp, err := s.deliveryClient.TrackShipment(...)
    if err != nil {
        s.logger.Warn("Failed to fetch tracking: %v", err)
        return nil // Don't fail entire GetOrder
    }
    // Enrich order...
}
```

---

## 16. Performance Considerations

### 16.1 Distributed Locks (Redis)

**Purpose:** Prevent race conditions on inventory updates

```go
// Lock product before reservation
lockKey := fmt.Sprintf("product:lock:%d", productID)
acquired, err := redisClient.SetNX(ctx, lockKey, 1, 5*time.Second).Result()

if !acquired {
    return fmt.Errorf("product locked by another operation")
}
defer redisClient.Del(ctx, lockKey)

// Reserve stock
...
```

### 16.2 Transaction Handling

**Database Transactions:**
```go
// order_service_tx.go:34
err := orderRepo.WithTx(ctx, s.db, func(tx *sqlx.Tx) error {
    // Create order
    createdOrder, err = s.createOrderInTx(ctx, tx, order)

    // Reserve stock
    reservations, err = s.processOrderItemsInTx(ctx, tx, createdOrder, items)

    // Update totals
    s.calculateOrderTotals(ctx, createdOrder, storefront)
    s.updateOrderInTx(ctx, tx, createdOrder)

    // Clear cart
    s.clearCartInTx(ctx, tx, *req.CartID)

    return nil
})
```

### 16.3 Caching

**Redis Cache:**
- Cart items (TTL: 30 minutes)
- Product availability (TTL: 1 minute)
- Shipping rates (TTL: 1 hour)

---

## 17. Security

### 17.1 Access Control

**Authorization Checks:**
```go
// Order access: buyer or seller only
func (s *OrderService) GetOrderByID(ctx, orderID, userID) {
    order, err := s.orderRepo.GetByID(ctx, orderID)

    // Check if buyer
    if order.CustomerID != userID {
        // Check if seller (storefront owner)
        storefront, err := s.storefrontRepo.GetByID(ctx, order.StorefrontID)
        if err != nil || storefront.UserID != userID {
            return fmt.Errorf("access denied")
        }
    }

    return order, nil
}
```

### 17.2 Payment Security

- **PCI Compliance:** No card data stored (AllSecure handles)
- **Webhook Verification:** HMAC signature validation
- **Session Expiry:** Payment sessions expire in 30 minutes
- **Escrow:** Funds held until delivery + protection period
- **TLS/SSL:** All payment API calls over HTTPS

---

## 18. Monitoring & Observability

### 18.1 Metrics

```go
// Prometheus metrics (example)
orders_created_total
orders_confirmed_total
orders_cancelled_total{reason}
orders_refunded_total
payment_success_rate
delivery_avg_time_hours
shipment_tracking_updates_total
inventory_reservations_expired_total
```

### 18.2 Structured Logging

```go
// order_service.go
s.logger.Info("Order created (order_id: %d, user_id: %d, total: %s)",
    order.ID, userID, order.TotalAmount.String())

s.logger.Warn("User %d attempted to pay for order %d belonging to user %d",
    userID, orderID, order.CustomerID)

s.logger.Error("Failed to create shipment: %v (order_id: %d)", err, orderID)
```

---

## 19. Future Improvements

- [ ] Split payments (multiple items → multiple sellers)
- [ ] Partial refunds
- [ ] Order disputes & resolution system
- [ ] Auto-cancellation of unpaid orders (15 min timeout)
- [ ] SMS notifications for order status
- [ ] Return/exchange flow
- [ ] Gift orders with custom message
- [ ] Scheduled delivery time slots
- [ ] Multi-currency support
- [ ] Tax calculation for international orders
- [ ] Subscription/recurring orders
- [ ] Order templates (repeat previous orders)

---

## 20. References

**Code Locations:**
- Order models: `backend/internal/domain/models/storefront_order.go`
- Order service: `backend/internal/proj/orders/service/order_service.go`
- Order transactions: `backend/internal/proj/orders/service/order_service_tx.go`
- Payment handler: `backend/internal/proj/payments/handler/order_payment_handler.go`
- Delivery integration: `backend/internal/proj/delivery/grpcclient/`
- Inventory management: `backend/internal/proj/orders/service/inventory_manager.go`

**Related Documents:**
- [Payment Flow](./payment.md)
- [Delivery Integration](./delivery.md)
- [Inventory Management](./../databases/listings_db.md#inventory)
- [Authentication](./../flows/authentication.md)

---

**Document Status:** Complete ✅
**Coverage:** 100% of current implementation
**Last Reviewed:** 2025-12-21
**Version:** 2.1.0
