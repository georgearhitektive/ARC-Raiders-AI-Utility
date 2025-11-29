
using System;
using System.Diagnostics;
using System.Drawing;
using System.IO;
using System.Management;
using System.Text;
using System.Windows.Forms;

namespace ARCUtility
{
    public partial class MainForm : Form
    {
        // Paths
        private readonly string configPath =
            Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData) +
            @"\PioneerGame\Saved\Config\WindowsClient\GameUserSettings.ini";

        private readonly string backupPath =
            Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
            @"PioneerGame\Saved\Config\WindowsClient\GameUserSettings_backup.ini");

        public MainForm()
        {
            InitializeComponent();
            CreateUI();
        }

        // ------------------------------
        // UI ELEMENTS
        // ------------------------------
        ComboBox cbPreset;
        CheckBox cbRTX;
        CheckBox cbNetLagFix;
        CheckBox cbOptimize;
        Button btnRun;
        Button btnRollback;
        TextBox txtLog;

        // ------------------------------
        // CREATE MODERN UI
        // ------------------------------
        private void CreateUI()
        {
            this.Text = "ARC RAIDERS – System Utility (.NET Edition)";
            this.Size = new Size(760, 700);
            this.FormBorderStyle = FormBorderStyle.FixedSingle;
            this.MaximizeBox = false;
            this.BackColor = Color.FromArgb(24, 24, 24);

            // TITLE PANEL
            Panel pnlTitle = new Panel
            {
                Size = new Size(this.Width, 70),
                Location = new Point(0, 0),
                BackColor = Color.FromArgb(40, 40, 40)
            };
            Label lblTitle = new Label
            {
                Text = "ARC Raiders - System Tools",
                Font = new Font("Segoe UI Semibold", 22, FontStyle.Bold),
                ForeColor = Color.White,
                AutoSize = true,
                Location = new Point(200, 15)
            };
            pnlTitle.Controls.Add(lblTitle);
            Controls.Add(pnlTitle);

            // OPTIONS PANEL
            Panel pnlOptions = new Panel
            {
                Size = new Size(720, 250),
                Location = new Point(20, 90),
                BackColor = Color.FromArgb(35, 35, 35),
                BorderStyle = BorderStyle.FixedSingle
            };
            Controls.Add(pnlOptions);

            cbRTX = CreateCheckBox("Automatic RTX detection + ini setup", 20, 20);
            pnlOptions.Controls.Add(cbRTX);

            cbNetLagFix = CreateCheckBox("Reduce Internet lag (TCP Optimizer + DNS + QoS off)", 20, 60);
            pnlOptions.Controls.Add(cbNetLagFix);

            cbOptimize = CreateCheckBox("ARC Raiders – full game optimization", 20, 100);
            pnlOptions.Controls.Add(cbOptimize);

            Label lblPreset = new Label
            {
                Text = "Graphics Preset:",
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 12),
                Location = new Point(20, 150),
                AutoSize = true
            };
            pnlOptions.Controls.Add(lblPreset);

            cbPreset = new ComboBox
            {
                Location = new Point(160, 147),
                Size = new Size(150, 28),
                Font = new Font("Segoe UI", 11),
                DropDownStyle = ComboBoxStyle.DropDownList,
                BackColor = Color.FromArgb(50, 50, 50),
                ForeColor = Color.White,
                FlatStyle = FlatStyle.Flat
            };
            cbPreset.Items.AddRange(new string[] { "Low", "Medium", "High", "Epic" });
            cbPreset.SelectedIndex = 2;
            pnlOptions.Controls.Add(cbPreset);

            // RUN BUTTON
            btnRun = CreateButton("▶ Run Selected Tasks", 40, 360, Color.FromArgb(255, 111, 97));
            btnRun.Click += RunTasks;
            Controls.Add(btnRun);

            // ROLLBACK BUTTON
            btnRollback = CreateButton("⟲ Safe Mode Rollback", 310, 360, Color.FromArgb(255, 111, 97));
            btnRollback.Click += Rollback;
            Controls.Add(btnRollback);

            // LOG PANEL
            Panel pnlLog = new Panel
            {
                Size = new Size(720, 240),
                Location = new Point(20, 430),
                BackColor = Color.FromArgb(40, 40, 40),
                BorderStyle = BorderStyle.FixedSingle
            };
            txtLog = new TextBox
            {
                Multiline = true,
                ScrollBars = ScrollBars.Vertical,
                Dock = DockStyle.Fill,
                Font = new Font("Consolas", 10),
                ForeColor = Color.White,
                BackColor = Color.FromArgb(24, 24, 24),
                ReadOnly = true
            };
            pnlLog.Controls.Add(txtLog);
            Controls.Add(pnlLog);
        }

        private CheckBox CreateCheckBox(string text, int x, int y)
        {
            return new CheckBox
            {
                Text = text,
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 12),
                Location = new Point(x, y),
                AutoSize = true
            };
        }

        private Button CreateButton(string text, int x, int y, Color baseColor)
        {
            Button btn = new Button
            {
                Text = text,
                Font = new Font("Segoe UI", 12, FontStyle.Bold),
                Size = new Size(250, 50),
                Location = new Point(x, y),
                BackColor = baseColor,
                ForeColor = Color.White,
                FlatStyle = FlatStyle.Flat
            };
            btn.FlatAppearance.BorderSize = 0;
            btn.MouseEnter += (s, e) => btn.BackColor = ControlPaint.Light(baseColor);
            btn.MouseLeave += (s, e) => btn.BackColor = baseColor;
            return btn;
        }

        // ------------------------------
        // LOG
        // ------------------------------
        private void Log(string message)
        {
            if (txtLog.InvokeRequired)
            {
                txtLog.Invoke(new Action(() => AppendLog(message)));
            }
            else
            {
                AppendLog(message);
            }
        }

        private void AppendLog(string message)
        {
            txtLog.AppendText(message + Environment.NewLine);
            txtLog.SelectionStart = txtLog.Text.Length;
            txtLog.ScrollToCaret();
        }

        // ------------------------------
        // RTX DETECTION + INI MODIFIER
        // ------------------------------
        private void ApplyRTXDetection()
        {
            Log("[RTX] Detecting GPU DXR / RayTracing capability...");

            bool supportsRTX = DetectDxrSupport();
            Log("GPU supports RTX: " + (supportsRTX ? "YES" : "NO"));

            string localApp = Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData);
            string configFile = Path.Combine(localApp, @"PioneerGame\Saved\Config\WindowsClient\GameUserSettings.ini");

            // Backup létrehozása
            try
            {
                Directory.CreateDirectory(Path.GetDirectoryName(backupPath));
                File.Copy(configFile, backupPath, true);
                Log("[RTX] Backup created at: " + backupPath);
            }
            catch (Exception ex)
            {
                Log("[RTX] Backup creation FAILED: " + ex.Message);
            }

            if (!File.Exists(configFile))
            {
                Log("[RTX] GameUserSettings.ini NOT FOUND!");
                return;
            }

            int screenX = Screen.PrimaryScreen.Bounds.Width;
            int screenY = Screen.PrimaryScreen.Bounds.Height;
            Log($"[RTX] Monitor resolution detected: {screenX}x{screenY}");

            string preset = cbPreset.SelectedItem.ToString();
            var ini = GenerateOptimizedIni(supportsRTX, screenX, screenY, preset);
            File.WriteAllText(configFile, ini, Encoding.UTF8);

            Log("[RTX] New optimized INI written successfully.");
            Log("✔ RTX Automatic Setup + INI generation completed.");
        }


        private bool DetectDxrSupport()
        {
            try
            {
                using var searcher = new ManagementObjectSearcher("select * from Win32_VideoController");
                foreach (var gpu in searcher.Get())
                {
                    string name = gpu["Name"].ToString().ToLower();

                    if (name.Contains("rtx"))
                        return true;

                    if (name.Contains("rx 6") || name.Contains("rx 7"))
                        return true;
                }
            }
            catch { }

            return false;
        }

        private string GenerateOptimizedIni(bool rtxSupported, int resX, int resY, string preset)
        {
            int viewDist = 2, texture = 2, foliage = 1, shadow = 1, aa = 2, effects = 1, post = 1, gi = 1, reflection = 1, shading = 1, resQuality = 80;

            switch (preset)
            {
                case "Low":
                    viewDist = 1; texture = 1; foliage = 0; shadow = 0; aa = 1;
                    effects = 0; post = 0; gi = 0; reflection = 0; shading = 0; resQuality = 70;
                    break;
                case "Medium":
                    viewDist = 2; texture = 2; foliage = 1; shadow = 1; aa = 2;
                    effects = 1; post = 1; gi = 1; reflection = 1; shading = 1; resQuality = 85;
                    break;
                case "High":
                    viewDist = 3; texture = 3; foliage = 2; shadow = 2; aa = 3;
                    effects = 2; post = 2; gi = 1; reflection = 2; shading = 1; resQuality = 100;
                    break;
                case "Epic":
                    viewDist = 4; texture = 4; foliage = 3; shadow = 3; aa = 4;
                    effects = 3; post = 3; gi = 2; reflection = 3; shading = 2; resQuality = 120;
                    break;
            }

            var sb = new StringBuilder();
            sb.AppendLine("[/Script/Engine.GameUserSettings]");
            sb.AppendLine("bUseDesiredScreenHeight=False");
            sb.AppendLine();
            sb.AppendLine("[/Script/EmbarkUserSettings.EmbarkGameUserSettings]");
            sb.AppendLine("WindowPosY=-1");
            sb.AppendLine("bUseHDRDisplayOutput=False");
            sb.AppendLine($"WindowedResolutionSizeX={resX}");

            if (rtxSupported)
            {
                sb.AppendLine("RTXGIQuality=DynamicHigh");
                sb.AppendLine("RTXGIResolutionQuality=2");
                sb.AppendLine("ResolutionScalingMethod=DLSS");
                sb.AppendLine("DLSSMode=DLAA");
            }
            else
            {
                sb.AppendLine("RTXGIQuality=Off");
                sb.AppendLine("RTXGIResolutionQuality=0");
                sb.AppendLine("ResolutionScalingMethod=FSR3");
                sb.AppendLine("DLSSMode=Off");
            }

            sb.AppendLine($"WindowedResolutionSizeY={resY}");
            sb.AppendLine($"ResolutionSizeY={resY}");
            sb.AppendLine($"ResolutionSizeX={resX}");
            sb.AppendLine("MotionBlurEnabled=False");
            sb.AppendLine("FrameRateLimit=0.000000");
            sb.AppendLine("NvReflexMode=Enabled");
            sb.AppendLine("bUseVSync=False");
            sb.AppendLine($"DesiredScreenWidth={resX}");
            sb.AppendLine($"DesiredScreenHeight={resY}");
            sb.AppendLine();
            sb.AppendLine("[ScalabilityGroups]");
            sb.AppendLine($"sg.ViewDistanceQuality={viewDist}");
            sb.AppendLine($"sg.TextureQuality={texture}");
            sb.AppendLine($"sg.FoliageQuality={foliage}");
            sb.AppendLine($"sg.ShadowQuality={shadow}");
            sb.AppendLine($"sg.AntiAliasingQuality={aa}");
            sb.AppendLine($"sg.EffectsQuality={effects}");
            sb.AppendLine($"sg.PostProcessQuality={post}");
            sb.AppendLine($"sg.GlobalIlluminationQuality={gi}");
            sb.AppendLine($"sg.ReflectionQuality={reflection}");
            sb.AppendLine($"sg.ShadingQuality={shading}");
            sb.AppendLine($"sg.ResolutionQuality={resQuality}");

            return sb.ToString();
        }

        // ------------------------------
        // NET LAG FIX
        // ------------------------------
        private void ApplyNetLagFix()
        {
            Log("[NET] Applying lag fix...");

            RunCmd("netsh int tcp set global autotuninglevel=normal");
            RunCmd("netsh int tcp set global ecncapability=disabled");
            RunCmd("netsh int tcp set global rss=enabled");
            RunCmd("netsh interface ipv4 set subinterface \"Ethernet\" mtu=1500 store=persistent");
            RunCmd("ipconfig /flushdns");
            RunCmd("netsh winsock reset");

            Log("[NET] Network optimization applied.");
        }

        // ------------------------------
        // GAME OPTIMIZATION
        // ------------------------------
        private void ApplyGameOptimization()
        {
            Log("[GAME] Applying ARC Raiders optimization...");
            RunCmd("wmic process where name=\"ARC.exe\" CALL setpriority 128");
            Log("[GAME] Game optimization done.");
        }

        private void RunCmd(string cmd)
        {
            try
            {
                Process.Start(new ProcessStartInfo("cmd.exe", "/c " + cmd)
                {
                    CreateNoWindow = true,
                    UseShellExecute = false
                });
            }
            catch (Exception ex)
            {
                Log("ERROR: " + ex.Message);
            }
        }

        // ------------------------------
        // SAFE MODE ROLLBACK
        // ------------------------------
        private void Rollback(object sender, EventArgs e)
        {
            txtLog.Clear();
            Log("Restoring original config...");

            if (File.Exists(backupPath))
            {
                File.Copy(backupPath, configPath, true);
                Log("✔ GameUserSettings.ini restored.");
            }
            else
            {
                Log("No backup found! Make sure to run at least once the RTX setup to create a backup.");
            }
        }

        // ------------------------------
        // RUN TASKS
        // ------------------------------
        private void RunTasks(object sender, EventArgs e)
        {
            txtLog.Clear();
            if (cbRTX.Checked) ApplyRTXDetection();
            if (cbNetLagFix.Checked) ApplyNetLagFix();
            if (cbOptimize.Checked) ApplyGameOptimization();
            Log("✔ All selected tasks completed.");
        }
    }
}

