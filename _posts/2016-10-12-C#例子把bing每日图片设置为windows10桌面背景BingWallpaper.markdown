
Wallpaper.cs 微软设置桌面背景的参考代码
====================================
```c#
﻿using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

using System.Runtime.InteropServices;
using System.ComponentModel;
using Microsoft.Win32;
using System.Drawing;
using System.IO;
using System.Drawing.Imaging;

namespace BingWallpaper
{

        public static class Wallpaper
        {
            /// <summary>
            /// Determine if .jpg files are supported as wallpaper in the current
            /// operating system. The .jpg wallpapers are not supported before
            /// Windows Vista.
            /// </summary>
            public static bool SupportJpgAsWallpaper
            {
                get
                {
                    return (Environment.OSVersion.Version >= new Version(6, 0));
                }
            }

            /// <summary>
            /// Determine if the fit and fill wallpaper styles are supported in
            /// the current operating system. The styles are not supported before
            /// Windows 7.
            /// </summary>
            public static bool SupportFitFillWallpaperStyles
            {
                get
                {
                    return (Environment.OSVersion.Version >= new Version(6, 1));
                }
            }

            /// <summary>
            /// Set the desktop wallpaper.
            /// </summary>
            /// <param name="path">Path of the wallpaper</param>
            /// <param name="style">Wallpaper style</param>
            public static void SetDesktopWallpaper(string path, WallpaperStyle style)
            {
                // Set the wallpaper style and tile.
                // Two registry values are set in the Control Panel\Desktop key.
                // TileWallpaper
                //  0: The wallpaper picture should not be tiled
                //  1: The wallpaper picture should be tiled
                // WallpaperStyle
                //  0:  The image is centered if TileWallpaper=0 or tiled if TileWallpaper=1
                //  2:  The image is stretched to fill the screen
                //  6:  The image is resized to fit the screen while maintaining the aspect
                //      ratio. (Windows 7 and later)
                //  10: The image is resized and cropped to fill the screen while
                //      maintaining the aspect ratio. (Windows 7 and later)
                RegistryKey key = Registry.CurrentUser.OpenSubKey(@"Control Panel\Desktop", true);

                switch (style)
                {
                    case WallpaperStyle.Tile:
                        key.SetValue(@"WallpaperStyle", "0");
                        key.SetValue(@"TileWallpaper", "1");
                        break;
                    case WallpaperStyle.Center:
                        key.SetValue(@"WallpaperStyle", "0");
                        key.SetValue(@"TileWallpaper", "0");
                        break;
                    case WallpaperStyle.Stretch:
                        key.SetValue(@"WallpaperStyle", "2");
                        key.SetValue(@"TileWallpaper", "0");
                        break;
                    case WallpaperStyle.Fit: // (Windows 7 and later)
                        key.SetValue(@"WallpaperStyle", "6");
                        key.SetValue(@"TileWallpaper", "0");
                        break;
                    case WallpaperStyle.Fill: // (Windows 7 and later)
                        key.SetValue(@"WallpaperStyle", "10");
                        key.SetValue(@"TileWallpaper", "0");
                        break;
                }

                key.Close();

                // If the specified image file is neither .bmp nor .jpg, - or -
                // if the image is a .jpg file but the operating system is Windows Server
                // 2003 or Windows XP/2000 that does not support .jpg as the desktop
                // wallpaper, convert the image file to .bmp and save it to the
                // %appdata%\Microsoft\Windows\Themes folder.
                string ext = Path.GetExtension(path);
                if ((!ext.Equals(".bmp", StringComparison.OrdinalIgnoreCase) &&
                    !ext.Equals(".jpg", StringComparison.OrdinalIgnoreCase))
                    ||
                    (ext.Equals(".jpg", StringComparison.OrdinalIgnoreCase) &&
                    !SupportJpgAsWallpaper))
                {
                    using (Image image = Image.FromFile(path))
                    {
                        path = String.Format(@"{0}\Microsoft\Windows\Themes\{1}.bmp",
                            Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData),
                            Path.GetFileNameWithoutExtension(path));
                        image.Save(path, ImageFormat.Bmp);
                    }
                }

                // Set the desktop wallpapaer by calling the Win32 API SystemParametersInfo
                // with the SPI_SETDESKWALLPAPER desktop parameter. The changes should
                // persist, and also be immediately visible.
                if (!SystemParametersInfo(SPI_SETDESKWALLPAPER, 0, path,
                    SPIF_UPDATEINIFILE | SPIF_SENDWININICHANGE))
                {
                    throw new Win32Exception();
                }
            }

            private const uint SPI_SETDESKWALLPAPER = 20;
            private const uint SPIF_UPDATEINIFILE = 0x01;
            private const uint SPIF_SENDWININICHANGE = 0x02;

            [DllImport("user32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
            [return: MarshalAs(UnmanagedType.Bool)]
            private static extern bool SystemParametersInfo(uint uiAction, uint uiParam,
                string pvParam, uint fWinIni);
        }

        public enum WallpaperStyle
        {
            Tile,
            Center,
            Stretch,
            Fit,
            Fill
        }

}

```



Form1.cs  使用webbrowser控件下载网页也解析背景图片 用webclient下载
================================================================
```c#
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Data;
using System.Drawing;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows.Forms;

using System.Net;
using System.Threading;
using System.Text.RegularExpressions;
using System.IO;


//  windows 10 锁屏界面的 Windows聚焦 图片在这个目录，也可以遍历今天的最新图片来设置背景？？
//  C:\Users\<用户>\AppData\Local\Packages\Microsoft.Windows.ContentDeliveryManager_cw5n1h2txyewy\LocalState\Assets


namespace BingWallpaper
{
    public partial class Form1 : Form
    {
        public Form1()
        {
            InitializeComponent();
        }

        private void Form1_Load(object sender, EventArgs e)
        {
            this.FormBorderStyle = FormBorderStyle.None;
            this.ShowInTaskbar = false;
            this.ShowIcon = false;
            this.Size = new Size(1, 1);
            this.Opacity = 0.01;
        
            try
            {
                DateTime t = File.GetCreationTime(@"D:\BingWallpaper.jpg");
                if (t.Day == DateTime.Now.Day)
                {
                    Application.Exit();
                }
            }
            catch
            {

            }
            //webBrowser1.Url = new Uri("http://cn.bing.com");
            webBrowser1.Navigate("http://cn.bing.com");
            timer1.Interval = 10000;
            timer1.Start();    
        }

        private void webBrowser1_DocumentCompleted(object sender, WebBrowserDocumentCompletedEventArgs e)
        {
            Thread myThread = new Thread(new ThreadStart(DownloadBingWallpaper));
            myThread.Start();
            BeginInvoke(new MethodInvoker(delegate
            {
                Hide();
            }));
        }

        private void DownloadBingWallpaper()
        {
            Thread.Sleep(5000);
            this.Invoke(new MethodInvoker(delegate
            {
                HtmlElement bgDiv = webBrowser1.Document.GetElementById("bgDiv");
                if (bgDiv == null)
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Data;
using System.Drawing;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows.Forms;

using System.Net;
using System.Threading;
using System.Text.RegularExpressions;
using System.IO;


//  windows 10 锁屏界面的 Windows聚焦 图片在这个目录，也可以遍历今天的最新图片来设置背景？？
//  C:\Users\<用户>\AppData\Local\Packages\Microsoft.Windows.ContentDeliveryManager_cw5n1h2txyewy\LocalState\Assets


namespace BingWallpaper
{
    public partial class Form1 : Form
    {
        public Form1()
        {
            InitializeComponent();
        }

        private void Form1_Load(object sender, EventArgs e)
        {
            this.FormBorderStyle = FormBorderStyle.None;
            this.ShowInTaskbar = false;
            this.ShowIcon = false;
            this.Size = new Size(1, 1);
            this.Opacity = 0.01;

            try
            {
                DateTime t = File.GetLastWriteTime(@"D:\BingWallpaper.jpg");
                if (t.Day == DateTime.Now.Day)
                {
                    Application.Exit();
                }
            }
            catch
            {

            }
            //webBrowser1.Url = new Uri("http://cn.bing.com");
            webBrowser1.Navigate("http://cn.bing.com");
            timer1.Interval = 60000;
            timer1.Start();    
        }

        private void webBrowser1_DocumentCompleted(object sender, WebBrowserDocumentCompletedEventArgs e)
        {
            Thread myThread = new Thread(new ThreadStart(DownloadBingWallpaper));
            myThread.Start();
            BeginInvoke(new MethodInvoker(delegate
            {
                Hide();
            }));
        }

        private void DownloadBingWallpaper()
        {
            Thread.Sleep(3000);
            this.Invoke(new MethodInvoker(delegate
            {
                HtmlElement bgDiv = webBrowser1.Document.GetElementById("bgDiv");
                if (bgDiv == null)
                {
                    goto END;
                }
                //string name = bgDiv.GetAttribute("name");
                string style = bgDiv.Style;
                
                string pattern = @"background-image:\s*url\(.(http:.*.jpg).\)";
                Match m = Regex.Match(style, pattern);
                if (!m.Success) {
                    goto END;
                }
                
                string imgUrl = m.Groups[1].Value;
                if (imgUrl.Length == 0)
                {
                    goto END;
                }

                WebClient webClient = new WebClient();

                try
                {
                    webClient.DownloadFile(imgUrl, @"D:\BingWallpaper.jpg");
                    Wallpaper.SetDesktopWallpaper(@"D:\BingWallpaper.jpg", WallpaperStyle.Fill);
                    Application.Exit();
                } catch {
                  
                }

            END:
                // this.Close();
                timer1.Stop();
                timer1.Start();
            }));
        }

        private void timer1_Tick(object sender, EventArgs e)
        {
            webBrowser1.Refresh();
            //webBrowser1.Navigate("http://cn.bing.com");
        }
    }
}

```




