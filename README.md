package com.example.maciez;

import javax.swing.*;
import javax.swing.border.CompoundBorder;
import javax.swing.border.EmptyBorder;
import javax.swing.border.LineBorder;
import javax.swing.border.MatteBorder;
import java.awt.*;
import java.awt.event.ActionEvent;
import java.awt.event.ActionListener;
import java.awt.event.KeyAdapter;
import java.awt.event.KeyEvent;
import java.awt.geom.AffineTransform;
import java.awt.geom.Ellipse2D;
import java.time.LocalTime;
import java.time.format.DateTimeFormatter;
import java.util.Locale;


class MeatTendernessDisplayApp {


    public enum TendernessCategory {
        MACIA("Macia", "Mercado de Exportação Premium / Steakhouses", new Color(0x2F, 0x7A, 0x4E)), MEDIA("Média", "Varejo Tradicional / Supermercados", new Color(0xB3, 0x79, 0x1E)), DURA("Dura", "Indústria de Processados (Hambúrgueres/Embutidos) ou Maturação", new Color(0xA1, 0x3A, 0x34));

        private final String label;
        private final String destination;
        private final Color inkColor;

        TendernessCategory(String label, String destination, Color inkColor) {
            this.label = label;
            this.destination = destination;
            this.inkColor = inkColor;
        }

        public String getLabel() {
            return label;
        }

        public String getDestination() {
            return destination;
        }

        public Color getInkColor() {
            return inkColor;
        }
    }

    public static final class TendernessResult {
        private final double measurement;
        private final TendernessCategory category;

        public TendernessResult(double measurement, TendernessCategory category) {
            this.measurement = measurement;
            this.category = category;
        }

        public double getMeasurement() {
            return measurement;
        }

        public TendernessCategory getCategory() {
            return category;
        }
    }

    public static final class TendernessClassifier {
        private final double softMediumThreshold;
        private final double mediumHardThreshold;

        public TendernessClassifier(double softMediumThreshold, double mediumHardThreshold) {
            if (!Double.isFinite(softMediumThreshold) || !Double.isFinite(mediumHardThreshold)) {
                throw new IllegalArgumentException("Thresholds devem ser números finitos.");
            }
            if (softMediumThreshold < 0 || mediumHardThreshold < 0) {
                throw new IllegalArgumentException("Thresholds devem ser não-negativos.");
            }
            if (softMediumThreshold >= mediumHardThreshold) {
                throw new IllegalArgumentException("softMediumThreshold deve ser menor que mediumHardThreshold.");
            }
            this.softMediumThreshold = softMediumThreshold;
            this.mediumHardThreshold = mediumHardThreshold;
        }

        public TendernessResult classify(double measurement) {
            if (!Double.isFinite(measurement)) {
                throw new IllegalArgumentException("Medição deve ser um número finito.");
            }
            if (measurement < 0) {
                throw new IllegalArgumentException("Medição deve ser não-negativa.");
            }
            TendernessCategory category;
            if (measurement <= softMediumThreshold) {
                category = TendernessCategory.MACIA;
            } else if (measurement <= mediumHardThreshold) {
                category = TendernessCategory.MEDIA;
            } else {
                category = TendernessCategory.DURA;
            }
            return new TendernessResult(measurement, category);
        }

        public double getSoftMediumThreshold() {
            return softMediumThreshold;
        }

        public double getMediumHardThreshold() {
            return mediumHardThreshold;
        }
    }


    private static final class ParseOutcome {
        enum Reason {OK, EMPTY, NEGATIVE, INVALID}

        final Reason reason;
        final double value;

        private ParseOutcome(Reason reason, double value) {
            this.reason = reason;
            this.value = value;
        }

        static ParseOutcome ok(double v) {
            return new ParseOutcome(Reason.OK, v);
        }

        static ParseOutcome fail(Reason r) {
            return new ParseOutcome(r, 0);
        }
    }

    private static ParseOutcome parseWeight(String raw) {
        String trimmed = raw.trim();
        if (trimmed.isEmpty()) {
            return ParseOutcome.fail(ParseOutcome.Reason.EMPTY);
        }


        String normalized = trimmed.replace(',', '.');

        double value;
        try {
            value = Double.parseDouble(normalized);
        } catch (NumberFormatException e) {

            return ParseOutcome.fail(ParseOutcome.Reason.INVALID);
        }

        if (!Double.isFinite(value)) {
            return ParseOutcome.fail(ParseOutcome.Reason.INVALID);
        }
        if (value < 0) {
            return ParseOutcome.fail(ParseOutcome.Reason.NEGATIVE);
        }
        return ParseOutcome.ok(value);
    }


    private static final Color CASING = new Color(0xE9, 0xEB, 0xEE);
    private static final Color BEZEL = Color.WHITE;
    private static final Color BEZEL_LINE = new Color(0xD7, 0xDB, 0xE1);
    private static final Color LCD_BG = new Color(0xCF, 0xE3, 0xC9);
    private static final Color LCD_INK = new Color(0x1E, 0x3A, 0x26);
    private static final Color LCD_INK_DIM = new Color(0x4C, 0x6A, 0x52);
    private static final Color PAPER = new Color(0xFD, 0xFC, 0xF8);
    private static final Color PAPER_LINE = new Color(0xE3, 0xDD, 0xC9);
    private static final Color TEXT = new Color(0x24, 0x26, 0x2B);
    private static final Color TEXT_DIM = new Color(0x76, 0x7C, 0x87);
    private static final Color STEEL_DARK = new Color(0x56, 0x5C, 0x66);
    private static final Color DANGER = new Color(0xB8, 0x38, 0x2D);

    private static final Font FONT_DISPLAY = new Font("SansSerif", Font.BOLD, 42);
    private static final Font FONT_TITLE = new Font("SansSerif", Font.BOLD, 30);
    private static final Font FONT_BODY = new Font("SansSerif", Font.PLAIN, 14);
    private static final Font FONT_BODY_BOLD = new Font("SansSerif", Font.BOLD, 15);
    private static final Font FONT_MONO = new Font("Monospaced", Font.PLAIN, 12);
    private static final Font FONT_MONO_BOLD = new Font("Monospaced", Font.BOLD, 13);

    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> new MainFrame().setVisible(true));
    }


    private static final class MainFrame extends JFrame {

        private final TendernessClassifier classifier = new TendernessClassifier(2.5, 5.0);
        private int ticketCounter = 0;

        private final JTextField weightField = new JTextField();
        private final JLabel errorLabel = new JLabel(" ");
        private final StampPanel stampPanel = new StampPanel();
        private final JLabel ticketHeadLabel = new JLabel(" ");
        private final JTextArea sentenceArea = new JTextArea();
        private final JLabel metaLabel = new JLabel(" ");
        private final JPanel historyListPanel = new JPanel();
        private final JLabel historyEmptyLabel = new JLabel("Os carimbos aparecem aqui conforme você pesa os cortes.");

        MainFrame() {
            super("Carimbo de Maciez — Estação de Classificação");
            setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
            getContentPane().setBackground(CASING);
            setLayout(new BorderLayout(0, 18));
            ((JComponent) getContentPane()).setBorder(new EmptyBorder(22, 26, 22, 26));

            add(buildHeader(), BorderLayout.NORTH);
            add(buildCenter(), BorderLayout.CENTER);
            add(buildHistory(), BorderLayout.SOUTH);

            setMinimumSize(new Dimension(860, 640));
            pack();
            setLocationRelativeTo(null);
        }


        private JComponent buildHeader() {
            JPanel panel = new JPanel();
            panel.setOpaque(false);
            panel.setLayout(new BoxLayout(panel, BoxLayout.Y_AXIS));

            JLabel eyebrow = new JLabel("ESTAÇÃO DE CLASSIFICAÇÃO · CONTROLE DE QUALIDADE");
            eyebrow.setFont(FONT_MONO_BOLD);
            eyebrow.setForeground(STEEL_DARK);

            JLabel title = new JLabel("Carimbo de Maciez");
            title.setFont(FONT_TITLE);
            title.setForeground(TEXT);
            title.setBorder(new EmptyBorder(4, 0, 4, 0));

            JLabel subtitle = new JLabel("<html><body style='width:520px'>Pese o corte em quilos. O sistema recusa letras " + "e valores negativos e mostra o destino comercial na hora.</body></html>");
            subtitle.setFont(FONT_BODY);
            subtitle.setForeground(TEXT_DIM);

            panel.add(eyebrow);
            panel.add(title);
            panel.add(subtitle);
            panel.setBorder(new CompoundBorder(new MatteBorder(0, 0, 1, 0, BEZEL_LINE), new EmptyBorder(0, 0, 14, 0)));
            return panel;
        }


        private JComponent buildCenter() {
            JPanel center = new JPanel(new GridLayout(1, 2, 18, 0));
            center.setOpaque(false);
            center.add(buildScalePanel());
            center.add(buildTicketPanel());
            return center;
        }

        private JComponent buildScalePanel() {
            JPanel panel = panelCard();
            panel.setLayout(new BorderLayout(0, 12));

            panel.add(sectionLabel("BALANÇA"), BorderLayout.NORTH);

            JPanel content = new JPanel();
            content.setOpaque(false);
            content.setLayout(new BoxLayout(content, BoxLayout.Y_AXIS));

            content.add(buildLcdDisplay());
            content.add(Box.createVerticalStrut(4));

            errorLabel.setFont(FONT_MONO);
            errorLabel.setForeground(DANGER);
            errorLabel.setAlignmentX(Component.LEFT_ALIGNMENT);
            content.add(errorLabel);
            content.add(Box.createVerticalStrut(10));

            content.add(buildKeypad());
            content.add(Box.createVerticalStrut(10));
            content.add(buildWeighButton());
            content.add(Box.createVerticalStrut(14));
            content.add(buildRangesLegend());

            panel.add(content, BorderLayout.CENTER);
            return panel;
        }

        private JComponent buildLcdDisplay() {
            JPanel lcd = new JPanel(new BorderLayout(8, 2));
            lcd.setBackground(LCD_BG);
            lcd.setBorder(new CompoundBorder(new LineBorder(new Color(0, 0, 0, 40), 1, true), new EmptyBorder(12, 14, 10, 14)));
            lcd.setAlignmentX(Component.LEFT_ALIGNMENT);
            lcd.setMaximumSize(new Dimension(Integer.MAX_VALUE, 92));

            JLabel caption = new JLabel("PESO DO CORTE");
            caption.setFont(FONT_MONO);
            caption.setForeground(LCD_INK_DIM);

            weightField.setFont(FONT_DISPLAY);
            weightField.setForeground(LCD_INK);
            weightField.setBackground(LCD_BG);
            weightField.setBorder(BorderFactory.createEmptyBorder());
            weightField.setCaretColor(LCD_INK);
            weightField.addActionListener(e -> weigh());
            weightField.addKeyListener(new KeyAdapter() {
                @Override
                public void keyReleased(KeyEvent e) {
                    clearError();
                }
            });

            JLabel unit = new JLabel("kg");
            unit.setFont(FONT_BODY_BOLD);
            unit.setForeground(LCD_INK_DIM);

            JPanel row = new JPanel(new BorderLayout(6, 0));
            row.setOpaque(false);
            row.add(weightField, BorderLayout.CENTER);
            row.add(unit, BorderLayout.EAST);

            JPanel wrapper = new JPanel(new BorderLayout(0, 4));
            wrapper.setOpaque(false);
            wrapper.add(caption, BorderLayout.NORTH);
            wrapper.add(row, BorderLayout.CENTER);

            lcd.add(wrapper, BorderLayout.CENTER);
            return lcd;
        }

        private JComponent buildKeypad() {
            JPanel keypad = new JPanel(new GridLayout(4, 3, 8, 8));
            keypad.setOpaque(false);
            keypad.setAlignmentX(Component.LEFT_ALIGNMENT);
            keypad.setMaximumSize(new Dimension(Integer.MAX_VALUE, 190));

            String[] keys = {"7", "8", "9", "4", "5", "6", "1", "2", "3", ",", "0", "back"};
            for (String key : keys) {
                JButton btn = new JButton("back".equals(key) ? "\u232B" : key);
                btn.setFont(FONT_MONO_BOLD);
                btn.setBackground(BEZEL);
                btn.setForeground(TEXT);
                btn.setFocusPainted(false);
                btn.setBorder(new CompoundBorder(new LineBorder(BEZEL_LINE, 1, true), new EmptyBorder(10, 0, 10, 0)));
                btn.addActionListener((ActionEvent e) -> {
                    if ("back".equals(key)) {
                        String t = weightField.getText();
                        if (!t.isEmpty()) {
                            weightField.setText(t.substring(0, t.length() - 1));
                        }
                    } else {
                        weightField.setText(weightField.getText() + key);
                    }
                    clearError();
                    weightField.requestFocusInWindow();
                });
                keypad.add(btn);
            }
            return keypad;
        }

        private JComponent buildWeighButton() {
            JButton weighBtn = new JButton("CARIMBAR");
            weighBtn.setFont(FONT_MONO_BOLD);
            weighBtn.setForeground(Color.WHITE);
            weighBtn.setBackground(STEEL_DARK);
            weighBtn.setFocusPainted(false);
            weighBtn.setAlignmentX(Component.LEFT_ALIGNMENT);
            weighBtn.setMaximumSize(new Dimension(Integer.MAX_VALUE, 44));
            weighBtn.setBorder(new EmptyBorder(10, 0, 10, 0));
            weighBtn.addActionListener((ActionListener) e -> weigh());
            return weighBtn;
        }

        private JComponent buildRangesLegend() {
            JPanel legend = new JPanel();
            legend.setOpaque(false);
            legend.setLayout(new BoxLayout(legend, BoxLayout.Y_AXIS));
            legend.setAlignmentX(Component.LEFT_ALIGNMENT);
            legend.setBorder(new CompoundBorder(new MatteBorder(1, 0, 0, 0, BEZEL_LINE), new EmptyBorder(10, 0, 0, 0)));

            legend.add(legendRow(TendernessCategory.MACIA.getInkColor(), "Até 2,5 kg — Macia"));
            legend.add(legendRow(TendernessCategory.MEDIA.getInkColor(), "Acima de 2,5 até 5,0 kg — Média"));
            legend.add(legendRow(TendernessCategory.DURA.getInkColor(), "Acima de 5,0 kg — Dura"));
            return legend;
        }

        private JComponent legendRow(Color dot, String text) {
            JPanel row = new JPanel(new FlowLayout(FlowLayout.LEFT, 6, 3));
            row.setOpaque(false);
            row.setAlignmentX(Component.LEFT_ALIGNMENT);

            JComponent dotComp = new JComponent() {
                @Override
                protected void paintComponent(Graphics g) {
                    Graphics2D g2 = (Graphics2D) g.create();
                    g2.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);
                    g2.setColor(dot);
                    g2.fillOval(0, 0, 8, 8);
                    g2.dispose();
                }
            };
            dotComp.setPreferredSize(new Dimension(8, 8));

            JLabel label = new JLabel(text);
            label.setFont(FONT_MONO);
            label.setForeground(TEXT_DIM);

            row.add(dotComp);
            row.add(label);
            return row;
        }


        private JComponent buildTicketPanel() {
            JPanel panel = panelCard();
            panel.setLayout(new BorderLayout(0, 12));
            panel.add(sectionLabel("TICKET DE CLASSIFICAÇÃO"), BorderLayout.NORTH);

            JPanel paper = new JPanel(new BorderLayout(0, 10));
            paper.setBackground(PAPER);
            paper.setBorder(new CompoundBorder(new LineBorder(PAPER_LINE, 1, true), new EmptyBorder(16, 16, 16, 16)));

            ticketHeadLabel.setFont(FONT_MONO);
            ticketHeadLabel.setForeground(TEXT_DIM);

            stampPanel.setPreferredSize(new Dimension(220, 220));

            sentenceArea.setFont(FONT_BODY_BOLD);
            sentenceArea.setForeground(TEXT);
            sentenceArea.setLineWrap(true);
            sentenceArea.setWrapStyleWord(true);
            sentenceArea.setEditable(false);
            sentenceArea.setOpaque(false);
            sentenceArea.setFocusable(false);
            sentenceArea.setText(" ");

            metaLabel.setFont(FONT_MONO);
            metaLabel.setForeground(TEXT_DIM);

            JPanel details = new JPanel(new BorderLayout(0, 6));
            details.setOpaque(false);
            details.setBorder(new CompoundBorder(new MatteBorder(1, 0, 0, 0, PAPER_LINE), new EmptyBorder(10, 0, 0, 0)));
            details.add(sentenceArea, BorderLayout.NORTH);
            details.add(metaLabel, BorderLayout.SOUTH);

            paper.add(ticketHeadLabel, BorderLayout.NORTH);
            paper.add(stampPanel, BorderLayout.CENTER);
            paper.add(details, BorderLayout.SOUTH);

            panel.add(paper, BorderLayout.CENTER);
            return panel;
        }


        private static final class StampPanel extends JComponent {
            private TendernessResult result;

            void setResult(TendernessResult result) {
                this.result = result;
                repaint();
            }

            void clear() {
                this.result = null;
                repaint();
            }

            @Override
            protected void paintComponent(Graphics g) {
                Graphics2D g2 = (Graphics2D) g.create();
                g2.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);
                int w = getWidth();
                int h = getHeight();

                if (result == null) {
                    g2.setColor(TEXT_DIM);
                    g2.setFont(FONT_BODY);
                    String line1 = "Nenhum carimbo ainda.";
                    String line2 = "Pese um corte para começar.";
                    FontMetrics fm = g2.getFontMetrics();
                    g2.drawString(line1, (w - fm.stringWidth(line1)) / 2, h / 2 - 6);
                    g2.drawString(line2, (w - fm.stringWidth(line2)) / 2, h / 2 + 14);
                    g2.dispose();
                    return;
                }

                Color ink = result.getCategory().getInkColor();
                int cx = w / 2;
                int cy = h / 2;
                int r1 = Math.min(w, h) / 2 - 14;
                int r2 = (int) (r1 * 0.83);

                AffineTransform old = g2.getTransform();
                g2.translate(cx, cy);
                g2.rotate(Math.toRadians(-7));

                g2.setColor(ink);
                g2.setStroke(new BasicStroke(4f));
                g2.draw(new Ellipse2D.Double(-r1, -r1, r1 * 2, r1 * 2));
                g2.setStroke(new BasicStroke(1.5f));
                g2.draw(new Ellipse2D.Double(-r2, -r2, r2 * 2, r2 * 2));

                g2.setFont(FONT_DISPLAY);
                FontMetrics fm = g2.getFontMetrics();
                String label = result.getCategory().getLabel().toUpperCase(Locale.ROOT);
                int tw = fm.stringWidth(label);
                g2.drawString(label, -tw / 2, fm.getAscent() / 2 - 4);

                g2.setTransform(old);
                g2.dispose();
            }
        }


        private JComponent buildHistory() {
            JPanel panel = panelCard();
            panel.setLayout(new BorderLayout(0, 10));

            JPanel headRow = new JPanel(new BorderLayout());
            headRow.setOpaque(false);
            headRow.add(sectionLabel("ÚLTIMOS CARIMBOS"), BorderLayout.WEST);

            JButton clearBtn = new JButton("Limpar histórico");
            clearBtn.setFont(FONT_MONO);
            clearBtn.setForeground(TEXT_DIM);
            clearBtn.setBackground(BEZEL);
            clearBtn.setFocusPainted(false);
            clearBtn.setBorder(new CompoundBorder(new LineBorder(BEZEL_LINE, 1, true), new EmptyBorder(6, 12, 6, 12)));
            clearBtn.addActionListener(e -> clearHistory());
            headRow.add(clearBtn, BorderLayout.EAST);

            historyListPanel.setOpaque(false);
            historyListPanel.setLayout(new BoxLayout(historyListPanel, BoxLayout.Y_AXIS));

            historyEmptyLabel.setFont(FONT_MONO);
            historyEmptyLabel.setForeground(TEXT_DIM);
            historyEmptyLabel.setBorder(new EmptyBorder(6, 2, 6, 2));
            historyListPanel.add(historyEmptyLabel);

            JScrollPane scroll = new JScrollPane(historyListPanel);
            scroll.setBorder(BorderFactory.createEmptyBorder());
            scroll.setOpaque(false);
            scroll.getViewport().setOpaque(false);
            scroll.setPreferredSize(new Dimension(10, 170));
            scroll.getVerticalScrollBar().setUnitIncrement(16);

            panel.add(headRow, BorderLayout.NORTH);
            panel.add(scroll, BorderLayout.CENTER);
            return panel;
        }

        private void clearHistory() {
            historyListPanel.removeAll();
            historyListPanel.add(historyEmptyLabel);
            historyListPanel.revalidate();
            historyListPanel.repaint();
            stampPanel.clear();
            ticketHeadLabel.setText(" ");
            sentenceArea.setText(" ");
            metaLabel.setText(" ");
            ticketCounter = 0;
        }

        private JComponent historyRow(TendernessResult result) {
            JPanel row = new JPanel(new BorderLayout(12, 0));
            row.setOpaque(true);
            row.setBackground(BEZEL);
            row.setBorder(new CompoundBorder(new LineBorder(BEZEL_LINE, 1, true), new EmptyBorder(8, 12, 8, 12)));
            row.setMaximumSize(new Dimension(Integer.MAX_VALUE, 40));

            JComponent dot = new JComponent() {
                @Override
                protected void paintComponent(Graphics g) {
                    Graphics2D g2 = (Graphics2D) g.create();
                    g2.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);
                    g2.setColor(result.getCategory().getInkColor());
                    g2.fillOval(0, 2, 10, 10);
                    g2.dispose();
                }
            };
            dot.setPreferredSize(new Dimension(14, 14));

            JLabel weightLabel = new JLabel(String.format(Locale.US, "%.2fkg", result.getMeasurement()));
            weightLabel.setFont(FONT_MONO_BOLD);
            weightLabel.setForeground(TEXT);
            weightLabel.setPreferredSize(new Dimension(70, 18));

            JLabel destLabel = new JLabel(result.getCategory().getDestination());
            destLabel.setFont(FONT_MONO);
            destLabel.setForeground(TEXT_DIM);

            JLabel timeLabel = new JLabel(LocalTime.now().format(DateTimeFormatter.ofPattern("HH:mm")));
            timeLabel.setFont(FONT_MONO);
            timeLabel.setForeground(TEXT_DIM);

            JPanel left = new JPanel(new FlowLayout(FlowLayout.LEFT, 8, 0));
            left.setOpaque(false);
            left.add(dot);
            left.add(weightLabel);

            row.add(left, BorderLayout.WEST);
            row.add(destLabel, BorderLayout.CENTER);
            row.add(timeLabel, BorderLayout.EAST);
            return row;
        }


        private void weigh() {
            ParseOutcome outcome = parseWeight(weightField.getText());

            if (outcome.reason != ParseOutcome.Reason.OK) {
                switch (outcome.reason) {
                    case EMPTY:
                        showError("Digite um peso antes de carimbar.");
                        break;
                    case NEGATIVE:
                        showError("Peso não pode ser negativo. Informe um valor a partir de 0.");
                        break;
                    default:
                        showError("Peso inválido. Use apenas números, como 2.5 ou 2,5.");
                }
                weightField.requestFocusInWindow();
                return;
            }

            clearError();
            TendernessResult result = classifier.classify(outcome.value);
            ticketCounter++;

            ticketHeadLabel.setText(String.format(Locale.US, "Ticket #%04d", ticketCounter) + "        " + LocalTime.now().format(DateTimeFormatter.ofPattern("HH:mm")));
            stampPanel.setResult(result);
            sentenceArea.setText("A carne é " + result.getCategory().getLabel() + ". Mande para " + result.getCategory().getDestination() + ".");
            metaLabel.setText(String.format(Locale.US, "%.2f kg pesados   |   Faixa: %s", result.getMeasurement(), result.getCategory().getLabel()));

            if (historyListPanel.getComponentCount() > 0 && historyListPanel.getComponent(0) == historyEmptyLabel) {
                historyListPanel.remove(historyEmptyLabel);
            }
            historyListPanel.add(historyRow(result), 0);
            historyListPanel.revalidate();
            historyListPanel.repaint();

            weightField.setText("");
            weightField.requestFocusInWindow();
        }

        private void showError(String msg) {
            errorLabel.setText(msg);
        }

        private void clearError() {
            errorLabel.setText(" ");
        }


        private JPanel panelCard() {
            JPanel panel = new JPanel();
            panel.setBackground(BEZEL);
            panel.setBorder(new CompoundBorder(new LineBorder(BEZEL_LINE, 1, true), new EmptyBorder(16, 16, 16, 16)));
            return panel;
        }

        private JComponent sectionLabel(String text) {
            JLabel label = new JLabel(text);
            label.setFont(FONT_MONO_BOLD);
            label.setForeground(STEEL_DARK);
            label.setBorder(new EmptyBorder(0, 0, 6, 0));
            return label;
        }
    }
}
