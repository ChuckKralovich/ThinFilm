
import javax.swing.*;
import javax.swing.event.ChangeListener;
import java.awt.*;
import java.awt.geom.Path2D;

/**
* Thin Film Interference Simulation (AP Physics 2)
*
* Three-wavelength (RGB) model.  Incident light is white = red + green + blue.
*
* Two reflected beams per color:
*   R1 (off the FRONT surface):  amplitude A,
*                                phase shift phi_1 = pi if n_film > n1, else 0
*   R2 (off the BACK  surface):  amplitude A,
*                                phase shift phi_2 = pi if n2 > n_film, else 0,
*                                plus round-trip phase 4 pi n_film t / lambda
*
* Resultant Reflected = R1 + R2 (per color).
* Transmitted from material 2: amplitude per color = sqrt(I_T),
*                              with I_R = (1 + cos delta) / 2, I_T = 1 - I_R.
*
* R1 and R2 individually have the same amplitude for every wavelength, so they
* look equally bright regardless of color.  The Resultant Reflected and the
* Transmitted rows show the AP Physics 2 thin-film coloring effect because the
* round-trip phase depends on lambda.
*
* PHYSICS NOTE ON ANIMATION:
*   A wave's temporal frequency f (hence angular frequency omega = 2 pi f) is a
*   property of the source and is CONSERVED when the wave crosses a boundary.
*   What changes across a boundary is the wave's speed (v = c/n) and therefore
*   its wavelength (lambda_medium = lambda_vacuum / n), since f = v / lambda
*   must hold in every medium.
*
*   In vacuum, every color travels at c, so on screen all colors must scroll at
*   the SAME phase velocity.  We enforce this by choosing a single shared
*   on-screen vacuum phase velocity V_SCROLL (px per time-unit) and deriving
*   each color's omega from its own vacuum wavenumber:
*       k_vac(color) = 2 pi / (lambda_vacuum * pxPerNm)
*       omega(color) = V_SCROLL * k_vac(color)
*   That single omega is then reused in all three regions for that color, while
*   the spatial wavenumber k = n * k_vac differs per region (wavelength
*   compression inside denser media).  Phase velocity in a region is
*   omega / k = V_SCROLL / n, so light correctly slows inside the film and
*   inside material 2 -- exactly as v = c / n requires.
*/
@FunctionalInterface
interface WaveFn { double y(double x); }

public class ThinFilmInterference extends JFrame {

    /** Fixed RGB wavelengths in nm */
    static final double[] WL = {660.0, 540.0, 440.0};
    /** Pure RGB colors used to draw each channel and build the net swatch */
    static final Color[]  PURE = {
        new Color(255, 40, 40),
        new Color(40, 220, 70),
        new Color(60, 130, 255)
    };

    /** Pale tints for the three media regions */
    static final Color TINT_M1   = new Color(255, 252, 200);
    static final Color TINT_FILM = new Color(220, 215, 255);
    static final Color TINT_M2   = new Color(255, 215, 225);

    /** Amplitude coefficient for each reflected beam, in units of incident amplitude */
    static final double R_AMP_COEFF = 0.5;

    private final boolean[] enabled = {true, true, true};
    private final SimulationPanel sim = new SimulationPanel();
    private final Timer animTimer;
    private final JButton pauseButton = new JButton("\u23F8 Pause");

    public ThinFilmInterference() {
        setTitle("Thin Film Interference Simulation");
        setSize(1500, 1050);
        setDefaultCloseOperation(EXIT_ON_CLOSE);
        setLocationRelativeTo(null);
        setLayout(new BorderLayout());

        JPanel controls = new JPanel(new GridBagLayout());
        controls.setBorder(BorderFactory.createEmptyBorder(8, 12, 8, 12));
        GridBagConstraints gbc = new GridBagConstraints();
        gbc.insets = new Insets(3, 6, 3, 6);
        gbc.fill = GridBagConstraints.HORIZONTAL;

        // R/G/B multi-select
        JPanel rgbBox = new JPanel(new FlowLayout(FlowLayout.LEFT, 12, 0));
        String[] cbLabels = {"Red (660 nm)", "Green (540 nm)", "Blue (440 nm)"};
        for (int i = 0; i < 3; i++) {
            final int idx = i;
            JCheckBox cb = new JCheckBox(cbLabels[i], true);
            cb.setForeground(new Color(40, 40, 40));
            cb.addActionListener(e -> {
                enabled[idx] = cb.isSelected();
                sim.setEnabledColors(enabled);
                sim.repaint();
            });
            rgbBox.add(cb);
        }
        gbc.gridy = 0;
        gbc.gridx = 0; gbc.weightx = 0;
        controls.add(new JLabel("Show colors:"), gbc);
        gbc.gridx = 1; gbc.weightx = 1;
        gbc.gridwidth = 3;
        controls.add(rgbBox, gbc);
        gbc.gridwidth = 1;

        JSlider n1Slider     = new JSlider(100, 300, 100);
        JLabel  n1Value      = new JLabel("1.00");
        JSlider nFilmSlider  = new JSlider(100, 300, 150);
        JLabel  nFilmValue   = new JLabel("1.50");
        JSlider n2Slider     = new JSlider(100, 300, 100);
        JLabel  n2Value      = new JLabel("1.00");
        JSlider thicknessSlider = new JSlider(50, 1500, 280);
        JLabel  thicknessValue  = new JLabel("280 nm");
        JSlider speedSlider = new JSlider(0, 30, 12);
        JLabel  speedValue  = new JLabel("1.0x");

        addRow(controls, gbc, 1, "n\u2081  (left material):",   n1Slider,    n1Value);
        addRow(controls, gbc, 2, "n   (thin film):",            nFilmSlider, nFilmValue);
        addRow(controls, gbc, 3, "n\u2082  (right material):",  n2Slider,    n2Value);
        addRow(controls, gbc, 4, "Film thickness t (nm):",      thicknessSlider, thicknessValue);
        addRow(controls, gbc, 5, "Animation speed:",            speedSlider, speedValue);

        JPanel playback = new JPanel(new FlowLayout(FlowLayout.LEFT, 8, 0));
        JButton resetButton = new JButton("\u21BB Reset");
        playback.add(pauseButton);
        playback.add(resetButton);
        gbc.gridy = 6;
        gbc.gridx = 0; gbc.weightx = 0;
        controls.add(new JLabel("Playback:"), gbc);
        gbc.gridx = 1; gbc.weightx = 1;
        gbc.gridwidth = 3;
        controls.add(playback, gbc);
        gbc.gridwidth = 1;

        JScrollPane controlsScroll = new JScrollPane(controls,
                JScrollPane.VERTICAL_SCROLLBAR_AS_NEEDED,
                JScrollPane.HORIZONTAL_SCROLLBAR_NEVER);
        controlsScroll.setBorder(null);
        controlsScroll.getVerticalScrollBar().setUnitIncrement(16);

        JSplitPane splitPane = new JSplitPane(JSplitPane.VERTICAL_SPLIT,
                                              sim, controlsScroll);
        splitPane.setOneTouchExpandable(true);
        splitPane.setResizeWeight(0.82);
        splitPane.setContinuousLayout(true);
        splitPane.setDividerSize(10);
        add(splitPane, BorderLayout.CENTER);

        ChangeListener cl = e -> {
            double n1 = n1Slider.getValue()    / 100.0;
            double nf = nFilmSlider.getValue() / 100.0;
            double n2 = n2Slider.getValue()    / 100.0;
            double t  = thicknessSlider.getValue();
            double sp = speedSlider.getValue() / 12.0;
            n1Value.setText(String.format("%.2f", n1));
            nFilmValue.setText(String.format("%.2f", nf));
            n2Value.setText(String.format("%.2f", n2));
            thicknessValue.setText((int) t + " nm");
            speedValue.setText(String.format("%.1fx", sp));
            sim.setParams(n1, nf, n2, t);
            sim.setSpeed(sp);
        };
        n1Slider.addChangeListener(cl);
        nFilmSlider.addChangeListener(cl);
        n2Slider.addChangeListener(cl);
        thicknessSlider.addChangeListener(cl);
        speedSlider.addChangeListener(cl);
        sim.setEnabledColors(enabled);
        cl.stateChanged(null);

        animTimer = new Timer(30, e -> { sim.advanceTime(); sim.repaint(); });
        animTimer.start();

        pauseButton.addActionListener(e -> togglePause());
        resetButton.addActionListener(e -> { sim.resetTime(); sim.repaint(); });

        JRootPane root = getRootPane();
        root.getInputMap(JComponent.WHEN_IN_FOCUSED_WINDOW)
            .put(KeyStroke.getKeyStroke("SPACE"), "togglePause");
        root.getActionMap().put("togglePause", new AbstractAction() {
            @Override
            public void actionPerformed(java.awt.event.ActionEvent e) { togglePause(); }
        });
    }

    private void togglePause() {
        if (animTimer.isRunning()) {
            animTimer.stop();
            pauseButton.setText("\u25B6 Play");
        } else {
            animTimer.start();
            pauseButton.setText("\u23F8 Pause");
        }
    }

    private void addRow(JPanel p, GridBagConstraints gbc, int row,
                        String label, JSlider slider, JLabel valueLabel) {
        gbc.gridy = row;
        gbc.gridx = 0; gbc.weightx = 0;
        p.add(new JLabel(label), gbc);
        gbc.gridx = 1; gbc.weightx = 1;
        p.add(slider, gbc);
        gbc.gridx = 2; gbc.weightx = 0;
        p.add(valueLabel, gbc);
    }

    /** Reflected intensity (0..1) for one wavelength using two-beam interference. */
    static double reflectedIntensity(double wl, double n1, double nFilm, double n2, double t) {
        double phi1 = (nFilm > n1) ? Math.PI : 0;
        double phi2 = (n2 > nFilm) ? Math.PI : 0;
        double delta = 4 * Math.PI * nFilm * t / wl + (phi2 - phi1);
        return (1 + Math.cos(delta)) / 2;
    }

    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> new ThinFilmInterference().setVisible(true));
    }
}

/**
* Main wave panel.  Draws:
*   - Inline swatch groups for Incident (label column), Resultant Reflected
*     (in material 1 region), and Transmitted from material 2 (in material 2 region)
*   - Big film header at top center
*   - Five wave rows:  Incident, R1, R2, Resultant Reflected, Transmitted
*   - Bottom readout block (intensities, phase shifts, 2nt)
*/
class SimulationPanel extends JPanel {

    public SimulationPanel() {
        setPreferredSize(new Dimension(1100, 760));
        setMinimumSize(new Dimension(700, 480));
    }

    private double time = 0;
    private double speed = 1.0;
    private double n1 = 1.0;
    private double nFilm = 1.5;
    private double n2 = 1.0;
    private double thickness = 280;
    private boolean[] enabled = {true, true, true};

    private final double pxPerNm = 0.25;
    private final int labelColWidth = 175;
    private final int filmLeftX     = 620;

    /**
     * Shared on-screen VACUUM phase velocity in px per time-unit.  Because every
     * color uses this same value to derive its omega, all colors scroll in
     * lockstep in vacuum (n = 1) -- as required since they all travel at c.
     * Inside a medium of index n the on-screen phase velocity becomes
     * V_SCROLL / n, so light visibly slows in the film and in material 2.
     */
    private static final double V_SCROLL = 0.5;

    public void advanceTime() { time += speed; }
    public void resetTime()   { time = 0; }
    public void setSpeed(double s) { this.speed = s; }
    public void setParams(double n1, double nf, double n2, double t) {
        this.n1 = n1; this.nFilm = nf; this.n2 = n2; this.thickness = t;
    }
    public void setEnabledColors(boolean[] e) { this.enabled = e.clone(); }

    @Override
    protected void paintComponent(Graphics g) {
        super.paintComponent(g);
        Graphics2D g2 = (Graphics2D) g;
        g2.setRenderingHint(RenderingHints.KEY_ANTIALIASING,
                            RenderingHints.VALUE_ANTIALIAS_ON);
        g2.setRenderingHint(RenderingHints.KEY_TEXT_ANTIALIASING,
                            RenderingHints.VALUE_TEXT_ANTIALIAS_ON);

        int width  = getWidth();
        int height = getHeight();
        int thicknessPx = (int) Math.round(thickness * pxPerNm);
        int filmRight   = filmLeftX + thicknessPx;

        // ---- Backgrounds ----
        g2.setColor(ThinFilmInterference.TINT_M1);
        g2.fillRect(labelColWidth, 0, filmLeftX - labelColWidth, height);
        g2.setColor(ThinFilmInterference.TINT_FILM);
        g2.fillRect(filmLeftX, 0, thicknessPx, height);
        g2.setColor(ThinFilmInterference.TINT_M2);
        g2.fillRect(filmRight, 0, width - filmRight, height);

        g2.setColor(new Color(228, 230, 240));
        g2.fillRect(0, 0, labelColWidth, height);
        g2.setColor(new Color(180, 184, 200));
        g2.setStroke(new BasicStroke(1f));
        g2.drawLine(labelColWidth, 0, labelColWidth, height);
        g2.setColor(new Color(60, 90, 140));
        g2.setStroke(new BasicStroke(1.5f));
        g2.drawLine(filmLeftX, 0, filmLeftX, height);
        g2.drawLine(filmRight, 0, filmRight, height);

        // ---- Per-color intensities ----
        double[] IR = new double[3];
        double[] IT = new double[3];
        for (int i = 0; i < 3; i++) {
            IR[i] = ThinFilmInterference.reflectedIntensity(
                    ThinFilmInterference.WL[i], n1, nFilm, n2, thickness);
            IT[i] = 1 - IR[i];
        }

        // ---- Top: medium headers and inline swatches ----
        // Material headers
        g2.setColor(new Color(40, 40, 40));
        g2.setFont(new Font("SansSerif", Font.BOLD, 14));
        drawCenteredString(g2,
                "Material 1   n\u2081 = " + String.format("%.2f", n1),
                (labelColWidth + filmLeftX) / 2, 22);
        drawCenteredString(g2,
                "Material 2   n\u2082 = " + String.format("%.2f", n2),
                (filmRight + width) / 2, 22);

        // Big film header
        g2.setFont(new Font("SansSerif", Font.BOLD, 22));
        drawCenteredString(g2,
                "Film   n = " + String.format("%.2f", nFilm),
                (filmLeftX + filmRight) / 2, 32);
        g2.setFont(new Font("SansSerif", Font.PLAIN, 12));
        g2.setColor(new Color(60, 90, 140));
        drawCenteredString(g2, "t = " + (int) thickness + " nm",
                (filmLeftX + filmRight) / 2, 52);

        // Swatch groups: Incident (label column), Reflected (M1), Transmitted (M2)
        int swatchY = 90;
        drawSwatchGroup(g2, "Incident",
                labelColWidth / 2, swatchY,
                new double[]{1, 1, 1});
        drawSwatchGroup(g2, "Resultant Reflected",
                (labelColWidth + filmLeftX) / 2, swatchY,
                IR);
        drawSwatchGroup(g2, "Transmitted from material 2",
                (filmRight + width) / 2, swatchY,
                IT);

        // ---- Layout: 4 wave rows ----
        int topMargin    = 160;
        int bottomBlockH = 120;
        int waveBandH    = Math.max(150, height - topMargin - bottomBlockH);
        int yIncident   = topMargin + waveBandH * 1 / 5;
        int yR1         = topMargin + waveBandH * 2 / 5;
        int yR2         = topMargin + waveBandH * 3 / 5;
        int yResultant  = topMargin + waveBandH * 4 / 5;

        int rowGap   = waveBandH / 5;
        int ampFull  = Math.max(12, Math.min(38, (int) (rowGap * 0.42)));

        // ---- Far-left row labels ----
        g2.setColor(Color.BLACK);
        g2.setFont(new Font("SansSerif", Font.BOLD, 13));
        g2.drawString("Incident \u2192", 10, yIncident - 8);
        g2.setFont(new Font("SansSerif", Font.PLAIN, 10));
        g2.drawString("white light",     10, yIncident + 6);

        g2.setFont(new Font("SansSerif", Font.BOLD, 13));
        g2.drawString("\u2190 R\u2081",  10, yR1 - 8);
        g2.setFont(new Font("SansSerif", Font.PLAIN, 10));
        g2.drawString("front-surface",   10, yR1 + 6);

        g2.setFont(new Font("SansSerif", Font.BOLD, 13));
        g2.drawString("\u2190 R\u2082",  10, yR2 - 8);
        g2.setFont(new Font("SansSerif", Font.PLAIN, 10));
        g2.drawString("back-surface",    10, yR2 + 6);

        g2.setFont(new Font("SansSerif", Font.BOLD, 13));
        g2.drawString("\u2190 Resultant",      10, yResultant - 8);
        g2.setFont(new Font("SansSerif", Font.PLAIN, 10));
        g2.drawString("R\u2081 + R\u2082",     10, yResultant + 6);

        // Axis lines
        g2.setColor(new Color(0, 0, 0, 55));
        g2.setStroke(new BasicStroke(1f));
        for (int y : new int[]{yIncident, yR1, yR2, yResultant}) {
            g2.drawLine(labelColWidth, y, width, y);
        }

        double phi1 = (nFilm > n1) ? Math.PI : 0;
        double phi2 = (n2 > nFilm) ? Math.PI : 0;

        // ---- Draw waves for each enabled color ----
        for (int c = 0; c < 3; c++) {
            if (!enabled[c]) continue;

            double wl = ThinFilmInterference.WL[c];

            // Vacuum wavenumber for this color (px^-1).  Spatial wavenumbers in
            // each medium scale by n: k_medium = n * k_vac (wavelength shrinks
            // by 1/n in a denser medium).
            final double kVac = 2 * Math.PI / (wl * pxPerNm);
            final double k1 = n1    * kVac;   // material 1
            final double kF = nFilm * kVac;   // film
            final double k3 = n2    * kVac;   // material 2

            // Temporal angular frequency: SAME in every region (frequency is
            // conserved across boundaries) and derived from the shared vacuum
            // phase velocity, so all colors scroll together in vacuum.
            final double omega = V_SCROLL * kVac;

            final double tNow = time;
            final int FL = filmLeftX;
            final int FR = filmRight;
            final int thickPx = thicknessPx;
            final double localPhi1 = phi1;
            final double localPhi2 = phi2;
            final int ampF = ampFull;

            Color base = ThinFilmInterference.PURE[c];
            Color waveC = new Color(base.getRed(), base.getGreen(), base.getBlue(), 210);
            g2.setColor(waveC);
            g2.setStroke(new BasicStroke(2.3f));

            // Row 1: Incident, continuous through all 3 regions.
            // Phase is matched at each boundary so the wave is continuous; the
            // spatial wavenumber changes (k1 -> kF -> k3) but omega does not.
            drawWave(g2, labelColWidth, FL, yIncident, x ->
                    ampF * Math.sin(k1 * x - omega * tNow));
            final double phFL = k1 * FL - omega * tNow;
            drawWave(g2, FL, FR, yIncident, x ->
                    ampF * Math.sin(phFL + kF * (x - FL)));
            final double phFR = phFL + kF * thickPx;
            drawWave(g2, FR, width, yIncident, x ->
                    ampF * Math.sin(phFR + k3 * (x - FR)));

            // Reflection amplitude per beam (in pixel units)
            final double aRef = ampF * ThinFilmInterference.R_AMP_COEFF;
            // Round-trip phase R2 picks up inside the film (down and back at kF)
            final double roundTrip = 2 * kF * thickPx;

            // Row 2: R1 -- reflects off FRONT surface, travels left in material 1
            //   y(x,t) = aRef * sin( k1*(2 FL - x) - omega t + phi1 )
            drawWave(g2, labelColWidth, FL, yR1, x ->
                    aRef * Math.sin(k1 * (2 * FL - x) - omega * tNow + localPhi1));

            // Row 3: R2 -- enters film, reflects off BACK surface, travels back, exits
            //   inside film, leftward, with kF
            //   outside film, leftward, with k1
            drawWave(g2, FL, FR, yR2, x -> {
                double pBack = k1 * FL + kF * thickPx - omega * tNow;
                return aRef * Math.sin(pBack + localPhi2 + kF * (FR - x));
            });
            drawWave(g2, labelColWidth, FL, yR2, x ->
                    aRef * Math.sin(k1 * (2 * FL - x) - omega * tNow
                                    + localPhi2 + roundTrip));

            // Row 4: Resultant Reflected = R1 + R2 (only left of film)
            drawWave(g2, labelColWidth, FL, yResultant, x -> {
                double basePh = k1 * (2 * FL - x) - omega * tNow;
                return aRef * Math.sin(basePh + localPhi1)
                     + aRef * Math.sin(basePh + localPhi2 + roundTrip);
            });
        }

        // ---- Bottom readout ----
        int yReadout = height - bottomBlockH + 20;
        g2.setColor(Color.BLACK);
        g2.setFont(new Font("SansSerif", Font.BOLD, 12));
        g2.drawString(String.format(
                "I_R(red) = %.2f     I_R(green) = %.2f     I_R(blue) = %.2f",
                IR[0], IR[1], IR[2]),
                labelColWidth + 10, yReadout);
        g2.drawString(String.format(
                "I_T(red) = %.2f     I_T(green) = %.2f     I_T(blue) = %.2f",
                IT[0], IT[1], IT[2]),
                labelColWidth + 10, yReadout + 22);
        g2.setFont(new Font("SansSerif", Font.PLAIN, 11));
        g2.setColor(new Color(60, 60, 60));
        g2.drawString(String.format(
                "Phase shifts:   R\u2081 = %s   (n_film %s n\u2081)         "
              + "R\u2082 = %s   (n\u2082 %s n_film)         2 n t = %.0f nm",
                (phi1 > 0 ? "\u03C0" : "0"),
                (nFilm > n1 ? ">" : "\u2264"),
                (phi2 > 0 ? "\u03C0" : "0"),
                (n2 > nFilm ? ">" : "\u2264"),
                2 * nFilm * thickness),
                labelColWidth + 10, yReadout + 46);

        g2.setFont(new Font("SansSerif", Font.PLAIN, 10));
        g2.setColor(new Color(140, 140, 140));
        g2.drawString("(Spacebar toggles pause   \u2022   in vacuum all colors scroll "
              + "together; light slows by 1/n inside each medium)",
                labelColWidth + 10, yReadout + 68);
    }

    /**
     * Draws a horizontal group of 4 swatches (R, G, B, net) with a title above,
     * centered on (cx, cy).  Intensities array has 3 entries (R, G, B), each 0..1.
     */
    private void drawSwatchGroup(Graphics2D g2, String title, int cx, int cy,
                                 double[] intensities) {
        int box = 22;
        int gap = 4;
        int totalW = 4 * box + 3 * gap;
        int startX = cx - totalW / 2;

        // Title
        g2.setColor(new Color(30, 30, 30));
        g2.setFont(new Font("SansSerif", Font.BOLD, 12));
        FontMetrics fmT = g2.getFontMetrics();
        // Wrap title onto 2 lines if it's long
        String[] lines = (title.length() > 14)
                ? splitInHalf(title) : new String[]{title};
        int titleY = cy - box / 2 - 6 - (lines.length - 1) * fmT.getHeight();
        for (String line : lines) {
            int tw = fmT.stringWidth(line);
            g2.drawString(line, cx - tw / 2, titleY);
            titleY += fmT.getHeight();
        }

        // Compute net (additive R+G+B)
        int totR = 0, totG = 0, totB = 0;
        int yBox = cy - box / 2;
        for (int c = 0; c < 3; c++) {
            double intensity = enabled[c]
                    ? Math.max(0, Math.min(1, intensities[c])) : 0;
            Color sw = new Color(
                (int) (ThinFilmInterference.PURE[c].getRed()   * intensity),
                (int) (ThinFilmInterference.PURE[c].getGreen() * intensity),
                (int) (ThinFilmInterference.PURE[c].getBlue()  * intensity)
            );
            int x = startX + c * (box + gap);
            g2.setColor(sw);
            g2.fillRect(x, yBox, box, box);
            g2.setColor(new Color(80, 80, 80));
            g2.drawRect(x, yBox, box, box);
            totR += sw.getRed();
            totG += sw.getGreen();
            totB += sw.getBlue();
        }
        int xNet = startX + 3 * (box + gap);
        g2.setColor(new Color(Math.min(255, totR),
                              Math.min(255, totG),
                              Math.min(255, totB)));
        g2.fillRect(xNet, yBox, box, box);
        g2.setColor(new Color(80, 80, 80));
        g2.drawRect(xNet, yBox, box, box);

        // Labels under boxes
        g2.setColor(new Color(50, 50, 50));
        g2.setFont(new Font("SansSerif", Font.BOLD, 10));
        FontMetrics fmL = g2.getFontMetrics();
        String[] labels = {"R", "G", "B", "net"};
        for (int i = 0; i < 4; i++) {
            int x = startX + i * (box + gap);
            int lw = fmL.stringWidth(labels[i]);
            g2.drawString(labels[i], x + (box - lw) / 2, yBox + box + 12);
        }
    }

    private String[] splitInHalf(String s) {
        int mid = s.length() / 2;
        int splitAt = -1;
        for (int d = 0; d < s.length(); d++) {
            if (mid + d < s.length() && s.charAt(mid + d) == ' ') { splitAt = mid + d; break; }
            if (mid - d >= 0 && s.charAt(mid - d) == ' ')         { splitAt = mid - d; break; }
        }
        if (splitAt < 0) return new String[]{s};
        return new String[]{ s.substring(0, splitAt), s.substring(splitAt + 1) };
    }

    private void drawWave(Graphics2D g2, int xStart, int xEnd, int yCenter, WaveFn f) {
        Path2D path = new Path2D.Double();
        boolean first = true;
        for (int x = xStart; x <= xEnd; x++) {
            double y = yCenter - f.y(x);
            if (first) { path.moveTo(x, y); first = false; }
            else        { path.lineTo(x, y); }
        }
        g2.draw(path);
    }

    private void drawCenteredString(Graphics2D g2, String s, int cx, int cy) {
        FontMetrics fm = g2.getFontMetrics();
        int w = fm.stringWidth(s);
        g2.drawString(s, cx - w / 2, cy);
    }
}
# ThinFilm
Thin film simulation
