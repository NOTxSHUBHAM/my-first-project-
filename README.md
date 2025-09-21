import javax.swing.*;
import java.awt.*;
import java.awt.event.*;
import java.sql.*;

public class FullFormFinder extends JFrame {
    private JTextField inputField;
    private JButton searchButton;
    private JButton clearButton;
    private JLabel resultLabel;
    private JPanel buttonPanel;
    private JPanel mainPanel;
    private JComboBox<String> historyComboBox;
    private JLabel statusLabel;

    public FullFormFinder() {
        initializeUI();
        setupEventHandlers();
    }

    private void initializeUI() {
        setTitle("Acronym Finder");
        setSize(600, 350);
        setDefaultCloseOperation(EXIT_ON_CLOSE);
        setLocationRelativeTo(null);
        
        try {
            UIManager.setLookAndFeel(UIManager.getSystemLookAndFeelClassName());
        } catch (Exception e) {
            e.printStackTrace();
        }

        mainPanel = new JPanel(new GridBagLayout());
        mainPanel.setBorder(BorderFactory.createEmptyBorder(20, 20, 10, 20));
        mainPanel.setBackground(new Color(240, 240, 245));
        
        GridBagConstraints gbc = new GridBagConstraints();
        gbc.insets = new Insets(10, 10, 10, 10);
        gbc.fill = GridBagConstraints.HORIZONTAL;
        gbc.weightx = 1.0;

        JLabel inputLabel = new JLabel("Enter Acronym/Short Form:");
        inputLabel.setFont(new Font("Segoe UI", Font.BOLD, 14));
        gbc.gridx = 0;
        gbc.gridy = 0;
        mainPanel.add(inputLabel, gbc);

        inputField = new JTextField();
        inputField.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        inputField.setPreferredSize(new Dimension(300, 30));
        gbc.gridy = 1;
        mainPanel.add(inputField, gbc);

        // History combo box
        historyComboBox = new JComboBox<>();
        historyComboBox.setFont(new Font("Segoe UI", Font.PLAIN, 12));
        historyComboBox.setBackground(Color.WHITE);
        historyComboBox.setEditable(false);
        historyComboBox.setEnabled(false);
        gbc.gridy = 2;
        mainPanel.add(historyComboBox, gbc);

        buttonPanel = new JPanel(new FlowLayout(FlowLayout.CENTER, 15, 0));
        searchButton = new JButton("Search");
        clearButton = new JButton("Clear");
        
        styleButton(searchButton, new Color(70, 130, 180));
        styleButton(clearButton, new Color(220, 53, 69));
        
        buttonPanel.add(searchButton);
        buttonPanel.add(clearButton);
        buttonPanel.setOpaque(false);
        gbc.gridy = 3;
        mainPanel.add(buttonPanel, gbc);

        resultLabel = new JLabel(" ", SwingConstants.CENTER);
        resultLabel.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        resultLabel.setBorder(BorderFactory.createEmptyBorder(10, 0, 0, 0));
        gbc.gridy = 4;
        mainPanel.add(resultLabel, gbc);

        // Status label
        statusLabel = new JLabel("Ready", SwingConstants.CENTER);
        statusLabel.setFont(new Font("Segoe UI", Font.ITALIC, 12));
        statusLabel.setForeground(Color.GRAY);
        gbc.gridy = 5;
        mainPanel.add(statusLabel, gbc);

        add(mainPanel);
    }

    private void styleButton(JButton button, Color bgColor) {
    button.setFont(new Font("Segoe UI", Font.BOLD, 14));
    button.setBackground(bgColor);
    button.setForeground(Color.WHITE);
    button.setFocusPainted(false);
    button.setBorder(BorderFactory.createCompoundBorder(
        BorderFactory.createLineBorder(new Color(0, 0, 0, 50)), 
        BorderFactory.createEmptyBorder(8, 20, 8, 20)
    ));
    button.setCursor(new Cursor(Cursor.HAND_CURSOR));
    
    // Add hover effect
    button.addMouseListener(new MouseAdapter() {
        @Override
        public void mouseEntered(MouseEvent e) {
            button.setBackground(bgColor.darker());
        }
        
        @Override
        public void mouseExited(MouseEvent e) {
            button.setBackground(bgColor);
        }
    });
}

    private void setupEventHandlers() {
        searchButton.addActionListener(e -> fetchFullForm());
        clearButton.addActionListener(e -> {
            inputField.setText("");
            resultLabel.setText(" ");
            inputField.requestFocus();
            updateStatus("Ready");
        });
        inputField.addActionListener(e -> fetchFullForm());
        
        // Add history selection handler
        historyComboBox.addActionListener(e -> {
            if (e.getSource() == historyComboBox && historyComboBox.getSelectedIndex() >= 0) {
                inputField.setText(historyComboBox.getSelectedItem().toString());
                fetchFullForm();
            }
        });
    }

    private void fetchFullForm() {
        String shortForm = inputField.getText().trim().toUpperCase();

        if (shortForm.isEmpty()) {
            showResult("Please enter an acronym/short form", Color.RED);
            updateStatus("Please enter a search term");
            return;
        }

        updateStatus("Searching for: " + shortForm);
        
        SwingWorker<Void, Void> worker = new SwingWorker<Void, Void>() {
            String resultText = "";
            Color resultColor = Color.BLACK;
            boolean success = false;

            @Override
            protected Void doInBackground() throws Exception {
                try (Connection conn = DatabaseConnection.getConnection()) {
                    String query = "SELECT full_form, description FROM full_forms WHERE short_form = ?";
                    PreparedStatement stmt = conn.prepareStatement(query);
                    stmt.setString(1, shortForm);
                    ResultSet rs = stmt.executeQuery();

                    if (rs.next()) {
                        String fullForm = rs.getString("full_form");
                        String description = rs.getString("description");
                        
                        resultText = "<html><div style='text-align: center;'>" +
                                   "<b>" + shortForm + ":</b> " + fullForm;
                        
                        if (description != null && !description.isEmpty()) {
                            resultText += "<br><br><i>" + description + "</i>";
                        }
                        
                        resultText += "</div></html>";
                        
                        resultColor = new Color(34, 139, 34);
                        success = true;
                        addToHistory(shortForm);
                    } else {
                        resultText = "No results found for: " + shortForm;
                        resultColor = Color.ORANGE;
                    }

                } catch (SQLException e) {
                    resultText = "<html><div style='text-align: center; color: red;'>" +
                                 "Database error!<br>" +
                                 "Please check:<br>" +
                                 "1. MySQL server is running<br>" +
                                 "2. Database credentials are correct<br>" +
                                 "3. Database and table exist</div></html>";
                    resultColor = Color.RED;
                }
                return null;
            }

            @Override
            protected void done() {
                showResult(resultText, resultColor);
                updateStatus(success ? "Found: " + shortForm : "No results for: " + shortForm);
            }
        };
        
        worker.execute();
    }

    private void addToHistory(String term) {
        ComboBoxModel<String> model = historyComboBox.getModel();
        for (int i = 0; i < model.getSize(); i++) {
            if (model.getElementAt(i).equals(term)) {
                return; // already in history
            }
        }
        
        historyComboBox.addItem(term);
        historyComboBox.setEnabled(true);
    }

    private void showResult(String text, Color color) {
        resultLabel.setText(text);
        resultLabel.setForeground(color);
    }
    
    private void updateStatus(String message) {
        statusLabel.setText(message);
    }

    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> {
            FullFormFinder app = new FullFormFinder();
            app.setVisible(true);
            
            // Test database connection at startup
            new Thread(() -> {
                try (Connection conn = DatabaseConnection.getConnection()) {
                    app.updateStatus("Database connection successful");
                } catch (SQLException e) {
                    app.updateStatus("Database connection failed");
                    app.showResult("<html><div style='text-align: center; color: red;'>" +
                                   "Failed to connect to database!<br>" +
                                   "Check your MySQL server and connection settings.</div></html>", 
                                   Color.RED);
                }
            }).start();
        });
    }
}

class DatabaseConnection {
    private static final String DB_URL = "jdbc:mysql://localhost:3306/acronym_db";
    private static final String USER = "acronym_user";
    private static final String PASS = "password";

    public static Connection getConnection() throws SQLException {
        try {
            Class.forName("com.mysql.cj.jdbc.Driver");
        } catch (ClassNotFoundException e) {
            throw new SQLException("MySQL JDBC Driver not found", e);
        }
        return DriverManager.getConnection(DB_URL, USER, PASS);
    }
}
