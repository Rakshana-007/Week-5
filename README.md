# Week-5

package com.rakshanamart.servlet;

import com.rakshanamart.util.DBConnection;

import javax.servlet.annotation.WebServlet;
import javax.servlet.http.*;
import java.io.IOException;
import java.math.BigDecimal;
import java.sql.*;
import java.util.*;

@WebServlet("/orders")
public class OrderServlet extends HttpServlet {

    @Override
    protected void doPost(HttpServletRequest request,
                          HttpServletResponse response)
            throws IOException {

        response.setContentType("application/json;charset=UTF-8");

        HttpSession session = request.getSession(false);
        if (session == null || session.getAttribute("userEmail") == null) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().print("{\"error\":\"Please log in first.\"}");
            return;
        }

        String email = (String) session.getAttribute("userEmail");
        String[] productIds = request.getParameterValues("productId");
        String[] quantities = request.getParameterValues("quantity");

        if (productIds == null || quantities == null ||
            productIds.length == 0 ||
            productIds.length != quantities.length) {
            response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
            response.getWriter().print("{\"error\":\"Invalid cart.\"}");
            return;
        }

        // Combine duplicate product IDs to avoid duplicate cart entries.
        Map<Integer, Integer> cart = new LinkedHashMap<>();

        try {
            for (int i = 0; i < productIds.length; i++) {
                int id = Integer.parseInt(productIds[i]);
                int qty = Integer.parseInt(quantities[i]);

                if (id <= 0 || qty <= 0) {
                    throw new IllegalArgumentException();
                }

                int combined = Math.addExact(
                        cart.getOrDefault(id, 0), qty);

                if (combined > 1000) {
                    throw new IllegalArgumentException();
                }
                cart.put(id, combined);
            }
        } catch (RuntimeException e) {
            response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
            response.getWriter().print("{\"error\":\"Invalid product or quantity.\"}");
            return;
        }

        Connection con = null;

        try {
            con = DBConnection.getConnection();
            con.setAutoCommit(false);

            int customerId;

            try (PreparedStatement ps = con.prepareStatement(
                    "SELECT id FROM customers WHERE email = ?")) {
                ps.setString(1, email);

                try (ResultSet rs = ps.executeQuery()) {
                    if (!rs.next()) {
                        con.rollback();
                        response.setStatus(
                                HttpServletResponse.SC_UNAUTHORIZED);
                        response.getWriter().print(
                                "{\"error\":\"Customer account not found.\"}");
                        return;
                    }
                    customerId = rs.getInt("id");
                }
            }

            BigDecimal total = BigDecimal.ZERO;

            // Lock and validate products before changing stock.
            for (Map.Entry<Integer, Integer> item : cart.entrySet()) {
                try (PreparedStatement ps = con.prepareStatement(
                        "SELECT price, stock FROM products WHERE id = ? FOR UPDATE")) {
                    ps.setInt(1, item.getKey());

                    try (ResultSet rs = ps.executeQuery()) {
                        if (!rs.next()) {
                            throw new IllegalArgumentException(
                                    "Product not found.");
                        }

                        BigDecimal price = rs.getBigDecimal("price");
                        int stock = rs.getInt("stock");
                        int qty = item.getValue();

                        if (stock < qty) {
                            throw new IllegalArgumentException(
                                    "Insufficient stock for product ID "
                                            + item.getKey());
                        }

                        total = total.add(
                                price.multiply(BigDecimal.valueOf(qty)));
                    }
                }
            }

            int orderId;

            try (PreparedStatement ps = con.prepareStatement(
                    "INSERT INTO orders (customer_id, total_amount, order_status) " +
                    "VALUES (?, ?, 'PENDING')",
                    Statement.RETURN_GENERATED_KEYS)) {

                ps.setInt(1, customerId);
                ps.setBigDecimal(2, total);
                ps.executeUpdate();

                try (ResultSet keys = ps.getGeneratedKeys()) {
                    if (!keys.next()) {
                        throw new SQLException("Could not create order.");
                    }
                    orderId = keys.getInt(1);
                }
            }

            for (Map.Entry<Integer, Integer> item : cart.entrySet()) {
                int productId = item.getKey();
                int qty = item.getValue();

                BigDecimal price;

                try (PreparedStatement ps = con.prepareStatement(
                        "SELECT price FROM products WHERE id = ?")) {
                    ps.setInt(1, productId);
                    try (ResultSet rs = ps.executeQuery()) {
                        if (!rs.next()) {
                            throw new SQLException("Product disappeared.");
                        }
                        price = rs.getBigDecimal("price");
                    }
                }

                try (PreparedStatement ps = con.prepareStatement(
                        "INSERT INTO order_items " +
                        "(order_id, product_id, quantity, price) VALUES (?, ?, ?, ?)")) {
                    ps.setInt(1, orderId);
                    ps.setInt(2, productId);
                    ps.setInt(3, qty);
                    ps.setBigDecimal(4, price);
                    ps.executeUpdate();
                }

                try (PreparedStatement ps = con.prepareStatement(
                        "UPDATE products SET stock = stock - ? " +
                        "WHERE id = ? AND stock >= ?")) {
                    ps.setInt(1, qty);
                    ps.setInt(2, productId);
                    ps.setInt(3, qty);

                    if (ps.executeUpdate() != 1) {
                        throw new SQLException(
                                "Stock changed; please try again.");
                    }
                }
            }

            con.commit();

            response.setStatus(HttpServletResponse.SC_CREATED);
            response.getWriter().printf(
                    Locale.US,
                    "{\"message\":\"Order created\",\"orderId\":%d,\"totalAmount\":\"%s\"}",
                    orderId, total.toPlainString());

        } catch (IllegalArgumentException e) {
            rollback(con);
            response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
            response.getWriter().print(
                    "{\"error\":\"" + jsonEscape(e.getMessage()) + "\"}");

        } catch (Exception e) {
            rollback(con);
            getServletContext().log("Order creation failed", e);
            response.setStatus(
                    HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
            response.getWriter().print(
                    "{\"error\":\"Unable to create order.\"}");

        } finally {
            if (con != null) {
                try {
                    con.setAutoCommit(true);
                    con.close();
                } catch (SQLException ignored) {
                    // Connection cleanup.
                }
            }
        }
    }

    private void rollback(Connection con) {
        if (con != null) {
            try {
                con.rollback();
            } catch (SQLException ignored) {
                // Preserve the original error.
            }
        }
    }

    private String jsonEscape(String value) {
        if (value == null) return "Invalid request";
        return value.replace("\\", "\\\\")
                    .replace("\"", "\\\"")
                    .replace("\n", "\\n")
                    .replace("\r", "\\r");
    }
}
