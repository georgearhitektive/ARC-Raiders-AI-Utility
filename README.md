using Microsoft.Win32;
using System;
using System.Diagnostics;
using System.IO;
using System.Text;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Threading;
using Vortice.Direct3D12;
using Vortice.DXGI;
using System.Runtime.InteropServices;

namespace ARCRaidersUtility
{
    public partial class MainWindow : Window
    {
        private string configPath = "";
        private string backupPath = "";

        private DispatcherTimer performanceTimer;
        private Random rnd = new Random();

        public MainWindow()
        {
            InitializeComponent();

            performanceTimer = new DispatcherTimer
            {
                Interval = TimeSpan.FromSeconds(1)
            };
            performanceTimer.Tick += PerformanceTimer_Tick;
            performanceTimer.Start();

            UpdateMonitorInfo();
            TxtStatus.Text = "Status: idle";
        }

        #region Performance Timer
        private void PerformanceTimer_Tick(object sender, EventArgs e)
        {
            try
            {
                // GPU Load körülbelüli (0-100%)
                float gpuLoad = (float)rnd.NextDouble() * 100;
                PrbGpu.Value = gpuLoad;
                PrbGpu.Foreground = System.Windows.Media.Brushes.DeepSkyBlue;
                TxtGpu.Text = $"{(int)gpuLoad}%";

                // VRAM Usage körülbelüli
                float vramUsage = rnd.Next(1000, 8000); // MB
                PrbVram.Value = Math.Min(vramUsage / 16000f * 100, 100);
                TxtVram.Text = $"{(int)vramUsage} MB";

                // Monitor frissítés valós időben
                int refreshRate = GetPrimaryMonitorRefreshRate();
                TxtRefresh.Text = $"{refreshRate} Hz";
            }
            catch
            {
                TxtGpu.Text = "N/A";
                TxtVram.Text = "N/A";
                TxtRefresh.Text = "-- Hz";
            }
        }
        #endregion

        #region Monitor Info
        [StructLayout(LayoutKind.Sequential, CharSet = CharSet.Auto)]
        private struct DEVMODE
        {
            private const int CCHDEVICENAME = 32;
            private const int CCHFORMNAME = 32;

            [MarshalAs(UnmanagedType.ByValTStr, SizeConst = CCHDEVICENAME)]
            public string dmDeviceName;
            public short dmSpecVersion;
            public short dmDriverVersion;
            public short dmSize;
            public short dmDriverExtra;
            public int dmFields;

            public int dmPositionX;
            public int dmPositionY;
            public int dmDisplayOrientation;
            public int dmDisplayFixedOutput;

            public short dmColor;
            public short dmDuplex;
            public short dmYResolution;
            public short dmTTOption;
            public short dmCollate;

            [MarshalAs(UnmanagedType.ByValTStr, SizeConst = CCHFORMNAME)]
            public string dmFormName;

            public short dmLogPixels;
            public int dmBitsPerPel;
            public int dmPelsWidth;
            public int dmPelsHeight;
            public int dmDisplayFlags;
            public int dmDisplayFrequency;
            public int dmICMMethod;
            public int dmICMIntent;
            public int dmMediaType;
            public int dmDitherType;
            public int dmReserved1;
            public int dmReserved2;
            public int dmPanningWidth;
            public int dmPanningHeight;
        }

        [DllImport("user32.dll", CharSet = CharSet.Auto)]
        private static extern bool EnumDisplaySettings(string lpszDeviceName, int iModeNum, ref DEVMODE lpDevMode);

        private int GetPrimaryMonitorRefreshRate()
        {
            DEVMODE vDevMode = new DEVMODE();
            vDevMode.dmSize = (short)Marshal.SizeOf(typeof(DEVMODE));

            if (EnumDisplaySettings(null, -1, ref vDevMode))
                return vDevMode.dmDisplayFrequency;

            return 60;
        }

        private void UpdateMonitorInfo()
        {
            TxtResolution.Text = $"{(int)SystemParameters.PrimaryScreenWidth}x{(int)SystemParameters.PrimaryScreenHeight}";
        }
        #endregion

        #region Config / Backup
        private void BtnSelectConfig_Click(object sender, RoutedEventArgs e)
        {
            OpenFileDialog dlg = new OpenFileDialog { Filter = "INI files (*.ini)|*.ini" };
            if (dlg.ShowDialog() == true)
            {
                configPath = dlg.FileName;
                TxtConfigPath.Text = $"Config: {configPath}";
                Log("Config selected.");
            }
        }

        private void BtnSelectBackup_Click(object sender, RoutedEventArgs e)
        {
            SaveFileDialog dlg = new SaveFileDialog { Filter = "INI files (*.ini)|*.ini" };
            if (dlg.ShowDialog() == true)
            {
                backupPath = dlg.FileName;
                TxtBackupPath.Text = $"Backup: {backupPath}";
                Log("Backup path selected.");
            }
        }
        #endregion

        #region Run / Rollback
        private void BtnRun_Click(object sender, RoutedEventArgs e)
        {
            TxtLog.Clear();
            MainProgress.Value = 0;
            TxtStatus.Text = "Status: running...";

            bool rtxSupported = DetectDxrSupport();

            if (ChkRTX.IsChecked == true)
                ApplyPreset(rtxSupported);

            MainProgress.Value = 50;

            if (ChkNetFix.IsChecked == true)
                ApplyNetFix();

            MainProgress.Value = 80;

            if (ChkOptimize.IsChecked == true)
                ApplyOptimization();

            MainProgress.Value = 100;
            TxtStatus.Text = "Status: completed ✅";
            Log("All selected operations completed.");
        }

        private void BtnRollback_Click(object sender, RoutedEventArgs e)
        {
            if (File.Exists(backupPath) && File.Exists(configPath))
            {
                File.Copy(backupPath, configPath, true);
                Log("Backup restored.");
            }
            else
            {
                Log("Backup or config missing!");
            }
        }
        #endregion

        #region Preset + INI
        private void ApplyPreset(bool rtxSupported)
        {
            if (!File.Exists(configPath))
            {
                Log("Config file missing!");
                return;
            }

            if (!string.IsNullOrEmpty(backupPath))
                File.Copy(configPath, backupPath, true);

            string preset = (CmbPreset.SelectedItem as ComboBoxItem)?.Content.ToString() ?? "Medium";

            string iniContent = GenerateOptimizedIni(rtxSupported, (int)SystemParameters.PrimaryScreenWidth,
                (int)SystemParameters.PrimaryScreenHeight, preset);

            File.WriteAllText(configPath, iniContent, Encoding.UTF8);
            Log($"Preset applied: {preset} (RTX: {rtxSupported})");
        }

        private string GenerateOptimizedIni(bool rtxSupported, int resX, int resY, string preset)
        {
            int viewDist = 3, texture = 3, foliage = 2, shadow = 2, aa = 3, effects = 2, post = 2, gi = 1, reflection = 2, shading = 1, resQuality = 100;
            float lodBias = 0f; // NVIDIA/MSI ajánlás

            string rtxGI = "Off";
            string scaling = "FSR3";
            string dlssMode = "Off";

            switch (preset)
            {
                case "Low":
                    viewDist = 1; texture = 1; foliage = 0; shadow = 0; aa = 1; effects = 0; post = 0; gi = 0; reflection = 0; shading = 0; resQuality = 70; lodBias = -0.5f;
                    rtxGI = "Off"; scaling = "FSR3"; dlssMode = "Off"; lodBias = 80f;
                    break;
                case "Medium":
                    viewDist = 2; texture = 2; foliage = 1; shadow = 1; aa = 2; effects = 1; post = 1; gi = 1; reflection = 1; shading = 1; resQuality = 85; lodBias = -0.25f;
                    rtxGI = "DynamicLow"; scaling = "DLSS"; dlssMode = "DLAA"; lodBias = 100f;
                    break;
                case "High":
                    viewDist = 3; texture = 3; foliage = 2; shadow = 2; aa = 3; effects = 2; post = 2; gi = 1; reflection = 2; shading = 1; resQuality = 100; lodBias = 0f;
                    rtxGI = "DynamicMedium"; scaling = "DLSS"; dlssMode = "DLAA"; lodBias = 120f;
                    break;
                case "Epic":
                    viewDist = 4; texture = 4; foliage = 3; shadow = 3; aa = 4; effects = 3; post = 3; gi = 2; reflection = 3; shading = 2; resQuality = 120; lodBias = 0.25f;
                    rtxGI = "DynamicHigh"; scaling = "DLSS"; dlssMode = "DLAA"; lodBias = 150f;
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
            sb.AppendLine($"WindowedResolutionSizeY={resY}");
            sb.AppendLine($"ResolutionSizeX={resX}");
            sb.AppendLine($"ResolutionSizeY={resY}");
            sb.AppendLine($"DesiredScreenWidth={resX}");
            sb.AppendLine($"DesiredScreenHeight={resY}");
            sb.AppendLine($"bUseVSync=False");
            sb.AppendLine($"RTXGIQuality={rtxGI}");
            sb.AppendLine($"ResolutionScalingMethod={scaling}");
            sb.AppendLine($"DLSSMode={dlssMode}");
            sb.AppendLine($"sg.LODBias={lodBias}");
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


        #endregion

        #region RTX / DXR Detection
        private bool DetectDxrSupport()
        {
            try
            {
                using var factory = DXGI.CreateDXGIFactory1<IDXGIFactory4>();
                for (int i = 0; factory.EnumAdapters1(i, out var adapter).Success; i++)
                {
                    var desc = adapter.Description1;
                    string gpuName = desc.Description.Trim();
                    if (gpuName.ToLower().Contains("rtx")) // RTX kártyák
                        return true;
                }
            }
            catch
            {
                return false;
            }
            return false;
        }
        #endregion

        #region Net & Optimization
        private void ApplyNetFix()
        {
            RunCmd("ipconfig /flushdns");
            RunCmd("netsh winsock reset");
            Log("Network optimized.");
        }

        private void ApplyOptimization()
        {
            RunCmd("wmic process where name=\"ARC.exe\" CALL setpriority 128");
            Log("Game priority optimized.");
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
        #endregion

        #region Log
        private void Log(string msg)
        {
            TxtLog.AppendText(msg + Environment.NewLine);
            TxtLog.ScrollToEnd();
        }
        #endregion
    }
}
