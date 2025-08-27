```markdown
# Foodex Fastfood Billing System – Detailed Implementation Plan

This plan outlines the step-by-step changes and new files required to build a self-contained fast food billing and order management web application with thermal printer support for 58mm receipts. It leverages our existing Next.js 15 project with TypeScript, Tailwind CSS, and shadcn/ui components.

---

## 1. Global and Foundational Changes

### 1.1 Update Global Styles
- **File:** `src/app/globals.css`
  - **Changes:**
    - Add responsive styling for mobile and desktop.
    - Insert print media queries to format receipt output for a 58mm thermal printer.  
    ```css
    @media print {
      body {
        width: 58mm;
        margin: 0;
      }
      /* Hide navigation and non-print content */
      .no-print {
        display: none;
      }
    }
    ```
  - **Error Handling/Best Practices:** Ensure that print styles gracefully fallback if the browser does not support print media queries.

---

## 2. Application Layout and Routing

### 2.1 Create Global Layout
- **File:** `src/app/layout.tsx`
  - **Changes:**
    - Create a new layout file to wrap all pages with a shared header and footer.
    - Wrap the application with an Order Context Provider (see Section 3).
    - Use Tailwind CSS to style a modern, clean navigation bar (text-based buttons only – no icons).
    - Example structure:
    ```tsx
    import React from "react";
    import { OrderProvider } from "@/context/OrderContext";

    export default function RootLayout({ children }: { children: React.ReactNode }) {
      return (
        <html lang="en">
          <head>
            <title>Foodex Fastfood Billing System</title>
          </head>
          <body className="bg-gray-50 text-gray-900">
            <OrderProvider>
              <header className="p-4 bg-white shadow-md flex justify-between items-center">
                <h1 className="text-xl font-bold">Foodex Billing</h1>
                <nav>
                  <a className="px-2" href="/">Home</a>
                  <a className="px-2" href="/receipt">Receipt</a>
                </nav>
              </header>
              <main className="p-4">
                {children}
              </main>
            </OrderProvider>
          </body>
        </html>
      );
    }
    ```

### 2.2 Create Main Order Page
- **File:** `src/app/page.tsx`
  - **Changes:**
    - Build the main interface to display the menu, order summary, and customer details form.
    - Import and render the following components:
      - **Menu Display:** Uses a grid layout to show menu items.
      - **Order Summary:** Displays currently selected items and pricing details.
      - **Customer Form:** Captures customer details (Name, Phone, Address) using React Hook Form and Zod.
    - Example snippet:
    ```tsx
    import React from "react";
    import MenuItemCard from "@/components/MenuItemCard";
    import OrderSummary from "@/components/OrderSummary";
    import CustomerForm from "@/components/CustomerForm";
    import { menuItems } from "@/lib/menuItems"; // A new file exporting the menu data

    export default function HomePage() {
      return (
        <div className="grid grid-cols-1 lg:grid-cols-2 gap-4">
          <section>
            <h2 className="text-2xl font-semibold mb-4">Menu</h2>
            <div className="grid grid-cols-2 sm:grid-cols-3 gap-4">
              {menuItems.map((item) => (
                <MenuItemCard key={item.id} item={item} />
              ))}
            </div>
          </section>
          <aside>
            <OrderSummary />
            <CustomerForm />
          </aside>
        </div>
      );
    }
    ```

### 2.3 Receipt Page for Thermal Printer
- **File:** `src/app/receipt/page.tsx`
  - **Changes:**
    - Render a dedicated page/component for the receipt output.
    - Use the `Receipt` component (see Section 4) to format order data for print.
    - Provide a button that triggers `window.print()` to send the receipt to the external thermal printer.
    - Example snippet:
    ```tsx
    import React from "react";
    import Receipt from "@/components/Receipt";

    export default function ReceiptPage() {
      return (
        <div>
          <Receipt />
          <button
            className="mt-4 px-4 py-2 bg-blue-600 text-white rounded"
            onClick={() => window.print()}
          >
            Print Receipt
          </button>
        </div>
      );
    }
    ```

---

## 3. State Management and Global Order Context

### 3.1 Create Order Context
- **File:** `src/context/OrderContext.tsx`
  - **Changes:**
    - Implement a React Context to manage the global state for the order (selected items, customer details, totals).
    - Define actions to add, remove, and update order items.
    - Wrap the application in the OrderProvider (used in layout.tsx).
    - Example:
    ```tsx
    import React, { createContext, useContext, useReducer } from "react";

    const OrderContext = createContext(null);

    const initialState = {
      items: [],
      customer: { name: "", phone: "", address: "" },
      discount: 0,
      deliveryCharge: 0,
    };

    function orderReducer(state, action) {
      switch (action.type) {
        case "ADD_ITEM":
          return { ...state, items: [...state.items, action.item] };
        case "REMOVE_ITEM":
          return { ...state, items: state.items.filter(item => item.id !== action.id) };
        case "UPDATE_CUSTOMER":
          return { ...state, customer: { ...state.customer, ...action.payload } };
        default:
          throw new Error(`Unhandled action type: ${action.type}`);
      }
    }

    export const OrderProvider = ({ children }) => {
      const [state, dispatch] = useReducer(orderReducer, initialState);
      return (
        <OrderContext.Provider value={{ state, dispatch }}>
          {children}
        </OrderContext.Provider>
      );
    };

    export const useOrder = () => {
      const context = useContext(OrderContext);
      if (!context) {
        throw new Error("useOrder must be used within an OrderProvider");
      }
      return context;
    };
    ```

---

## 4. Component Development

### 4.1 Menu Item Card Component
- **File:** `src/components/MenuItemCard.tsx`
  - **Changes:**
    - Render a card view with the item name, price, and an “Add” button.
    - Use Tailwind CSS to style a modern card with clean typography.
    - On clicking “Add”, call the dispatch from the Order Context.
    - Example:
    ```tsx
    import React from "react";
    import { useOrder } from "@/context/OrderContext";

    const MenuItemCard = ({ item }) => {
      const { dispatch } = useOrder();

      const handleAdd = () => {
        try {
          dispatch({ type: "ADD_ITEM", item });
        } catch (error) {
          console.error("Failed to add item", error);
        }
      };

      return (
        <div className="p-4 border rounded shadow-sm bg-white">
          <h3 className="text-lg font-medium">{item.name}</h3>
          <p className="mt-1 text-gray-700">${item.price.toFixed(2)}</p>
          <button
            className="mt-2 w-full py-1 bg-green-500 text-white rounded"
            onClick={handleAdd}
          >
            Add
          </button>
        </div>
      );
    };

    export default MenuItemCard;
    ```

### 4.2 Order Summary Component
- **File:** `src/components/OrderSummary.tsx`
  - **Changes:**
    - Display a list of items currently in the order and calculate the subtotal, discount, delivery charge, and total.
    - Include a "Remove" button for each item.
    - Use helper functions (e.g., from `src/lib/utils.ts`) for calculations.
    - Example:
    ```tsx
    import React from "react";
    import { useOrder } from "@/context/OrderContext";
    import { calculateTotals } from "@/lib/utils";

    const OrderSummary = () => {
      const { state, dispatch } = useOrder();
      const { subtotal, total } = calculateTotals(state.items, state.discount, state.deliveryCharge);

      return (
        <div className="p-4 border rounded bg-white">
          <h2 className="text-xl font-semibold">Order Summary</h2>
          <ul>
            {state.items.map((item) => (
              <li key={item.id} className="flex justify-between my-2">
                <span>{item.name} x1</span>
                <span>${item.price.toFixed(2)}</span>
                <button
                  className="text-red-500 ml-2"
                  onClick={() => dispatch({ type: "REMOVE_ITEM", id: item.id })}
                >
                  Remove
                </button>
              </li>
            ))}
          </ul>
          <div className="mt-4">
            <p>Subtotal: ${subtotal.toFixed(2)}</p>
            <p>Total: ${total.toFixed(2)}</p>
          </div>
        </div>
      );
    };

    export default OrderSummary;
    ```

### 4.3 Customer Details Form Component
- **File:** `src/components/CustomerForm.tsx`
  - **Changes:**
    - Build a form using React Hook Form and validate inputs with Zod.
    - Capture Name, Phone, and Address fields.
    - On submission, dispatch an "UPDATE_CUSTOMER" action.
    - Display inline error messages if validation fails.
    - Example:
    ```tsx
    import React from "react";
    import { useForm } from "react-hook-form";
    import { zodResolver } from "@hookform/resolvers/zod";
    import * as z from "zod";
    import { useOrder } from "@/context/OrderContext";

    const customerSchema = z.object({
      name: z.string().nonempty("Name is required"),
      phone: z.string().nonempty("Phone is required"),
      address: z.string().nonempty("Address is required"),
    });

    type CustomerFormData = z.infer<typeof customerSchema>;

    const CustomerForm = () => {
      const { dispatch } = useOrder();
      const { register, handleSubmit, formState: { errors } } = useForm<CustomerFormData>({
        resolver: zodResolver(customerSchema),
      });

      const onSubmit = (data: CustomerFormData) => {
        dispatch({ type: "UPDATE_CUSTOMER", payload: data });
      };

      return (
        <form onSubmit={handleSubmit(onSubmit)} className="mt-4 p-4 border rounded bg-white">
          <h2 className="text-xl font-semibold mb-2">Customer Details</h2>
          <div className="mb-3">
            <label className="block font-medium">Name:</label>
            <input {...register("name")} className="w-full p-2 border rounded" />
            {errors.name && <p className="text-red-500">{errors.name.message}</p>}
          </div>
          <div className="mb-3">
            <label className="block font-medium">Phone:</label>
            <input {...register("phone")} className="w-full p-2 border rounded" />
            {errors.phone && <p className="text-red-500">{errors.phone.message}</p>}
          </div>
          <div className="mb-3">
            <label className="block font-medium">Address:</label>
            <input {...register("address")} className="w-full p-2 border rounded" />
            {errors.address && <p className="text-red-500">{errors.address.message}</p>}
          </div>
          <button type="submit" className="mt-2 px-4 py-2 bg-blue-600 text-white rounded">
            Save Details
          </button>
        </form>
      );
    };

    export default CustomerForm;
    ```

### 4.4 Receipt Component for Thermal Printer
- **File:** `src/components/Receipt.tsx`
  - **Changes:**
    - Format the receipt in a clean, print-friendly layout optimized for 58mm paper width.
    - Display order items, cost breakdown, and customer details.
    - Use Tailwind CSS typography and spacing; ensure no icons/images are used.
    - Example:
    ```tsx
    import React from "react";
    import { useOrder } from "@/context/OrderContext";
    import { calculateTotals } from "@/lib/utils";

    const Receipt = () => {
      const { state } = useOrder();
      const { subtotal, total } = calculateTotals(state.items, state.discount, state.deliveryCharge);

      return (
        <div className="p-2">
          <h2 className="text-center font-bold">Foodex Receipt</h2>
          <div className="mt-2">
            <p>Customer: {state.customer.name}</p>
            <p>Phone: {state.customer.phone}</p>
            <p>Address: {state.customer.address}</p>
          </div>
          <hr className="my-2 border-dashed" />
          <div>
            {state.items.map((item) => (
              <div key={item.id} className="flex justify-between text-sm">
                <span>{item.name}</span>
                <span>${item.price.toFixed(2)}</span>
              </div>
            ))}
          </div>
          <hr className="my-2 border-dashed" />
          <div className="text-sm">
            <p>Subtotal: ${subtotal.toFixed(2)}</p>
            <p>Total: ${total.toFixed(2)}</p>
          </div>
        </div>
      );
    };

    export default Receipt;
    ```

---

## 5. Utility Functions

### 5.1 Calculation Helpers
- **File:** `src/lib/utils.ts`
  - **Changes:**
    - Add a helper function `calculateTotals` to compute the subtotal and total.
    - Apply any discount and delivery charge logic.
    - Example:
    ```ts
    export function calculateTotals(items = [], discount = 0, deliveryCharge = 0) {
      const subtotal = items.reduce((sum, item) => sum + item.price, 0);
      const total = subtotal - discount + deliveryCharge;
      return { subtotal, total };
    }
    ```

### 5.2 Menu Data Source
- **File:** `src/lib/menuItems.ts`
  - **Changes:**
    - Create an array of menu items with properties such as id, name, price, and category.
    - Example:
    ```ts
    export const menuItems = [
      { id: 1, name: "Burger", price: 5.99, category: "Burgers" },
      { id: 2, name: "Pizza", price: 8.99, category: "Pizza" },
      { id: 3, name: "Fries", price: 2.99, category: "Sides" },
      { id: 4, name: "Soda", price: 1.99, category: "Drinks" },
      // Add other items as needed…
    ];
    ```

---

## 6. Error Handling and Best Practices

- **Input/Form Errors:** Use Zod validation in the customer form and display inline error messages.
- **Context Errors:** Wrap dispatch calls in try/catch blocks to log errors.
- **Async & Operations:** Although simulation is used, if payment or external API operations are added later, handle errors via proper try/catch and display user-friendly messages.
- **Testing & Debugging:** Use browser console logging and in-browser testing (e.g., `npm run dev`) to verify all interactions and printing behavior.

---

## 7. Testing Workflow

- **Local Testing:**  
  - Run `npm run dev` and test each functionality: adding/removing menu items, customer form validation, receipt generation, and printing.
- **Thermal Printer Test:**  
  - Verify that the print preview (via `window.print()`) correctly formats within 58mm width using the added print CSS.

---

## Summary

- Created a new layout and routing structure in the app folder, including pages for the order interface and receipt printing.
- Developed global state management using a React Context (`OrderContext`) to manage order and customer data.
- Implemented core components: MenuItemCard, OrderSummary, CustomerForm, and Receipt, each styled with Tailwind CSS.
- Added utility functions for calculating totals and maintained error handling via validations and try/catch.
- Configured global print CSS for thermal printer-optimized receipt output.
- The system is self-contained with simulated payment processing and leverages best practices for modular React development.
