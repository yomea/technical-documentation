此文最早发表自我的博客园博文[《JAVA图形界面写个小桌面，内置简单小软件》](https://www.cnblogs.com/honger/p/5994885.html)

一、做个这样的效果，双击图标打开相应的应用
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/233ac6a1066b03e215aa5a75ba2ebcc5.png)
二、主界面类，使用JavaSwing的JDesktopPane类创建这个桌面

    package com.swing;
    
    import java.awt.BorderLayout;
    import java.awt.Dimension;
    import java.awt.Graphics2D;
    import java.awt.Rectangle;
    import java.awt.Toolkit;
    import java.awt.event.MouseAdapter;
    import java.awt.event.MouseEvent;
    import java.awt.image.BufferedImage;
    import java.io.File;
    import java.io.IOException;
    import java.util.Timer;
    import java.util.TimerTask;
    
    import javax.imageio.ImageIO;
    import javax.swing.ImageIcon;
    import javax.swing.JButton;
    import javax.swing.JDesktopPane;
    import javax.swing.JFileChooser;
    import javax.swing.JFrame;
    import javax.swing.JLabel;
    
    import com.swing.plane.PanelGame;
    import com.swing.sunModel.SunModel;
    /**
     * 获取文件的图标
        FileSystemView fileSystemView = FileSystemView.getFileSystemView();
        ImageIcon icon = (ImageIcon) fileSystemView.getSystemIcon(file);
        BufferedImage i = new BufferedImage(100, 100, BufferedImage.TYPE_INT_RGB);
        i.getGraphics().drawImage(icon.getImage(), 0, 0, null);
        File out = new File("src/lib/hello.png");
        try {
            ImageIO.write(i, "png", out);
        } catch (IOException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
     * @author may
     *
     */
    public class Desktop extends JFrame {
        
        
        private static final long serialVersionUID = 3899092629742973479L;
        
        private JDesktopPane desktop = null;//定义桌面面板
        private JLabel backgroundImage = null;//定义桌面背景
        private MouseOption mouseOption = new MouseOption();//鼠标监听对象
    
        public Desktop(String title) {
            super(title);
            Toolkit toolkit = Toolkit.getDefaultToolkit();
            //得到系统屏幕的大小
            Dimension dimension = toolkit.getScreenSize();
            //设置布局管理为BorderLayout
            this.setLayout(new BorderLayout());
            int width = (int)dimension.getWidth();
            int height = (int)dimension.getHeight() - 100;
            this.setSize(width, height);
            desktop = new JDesktopPane();
            backgroundImage = new JLabel();
            //创建一个空的的图片
            BufferedImage image = new BufferedImage(width, height, BufferedImage.TYPE_INT_RGB);
            Graphics2D g = image.createGraphics();
            BufferedImage ad = null;
            try {
                //读取背景图
                ad = ImageIO.read(this.getClass().getResource("/lib/rapeFlower.jpg"));
            } catch (IOException e) {
                
                e.printStackTrace();
            }
            //将背景图按比例缩放重新画到之前创建的空图片
            g.drawImage(ad, 0, 0, width, height, null);
            //转化为Icon类图片
            ImageIcon img = new ImageIcon(image);
            backgroundImage.setIcon(img);
            //设置存放背景图的背景标签的位置和大小
            backgroundImage.setBounds(new Rectangle(0, 0, width, height));
            
            //创建按钮
            JButton myCompute = new JButton();
            myCompute.setIcon(new ImageIcon(this.getClass().getResource("/lib/computer.png")));
            myCompute.setBounds(20, 20, 48, 48);
            //设置按钮为透明
            myCompute.setContentAreaFilled(false);
            //除去边框
            myCompute.setBorderPainted(false);
            //添加事件监听
            myCompute.addMouseListener(mouseOption );
            //设置它的文本标识
            myCompute.setText("compute");
            //添加到桌面，并且比设置它的层次，比背景图更高一个层次，否侧会被背景图遮住，看不见
            desktop.add(myCompute, Integer.MIN_VALUE + 1);
            
            JButton myNotebook= new JButton();
            myNotebook.setIcon(new ImageIcon(this.getClass().getResource("/lib/notebook.png")));
            myNotebook.setBounds(20, 88, 48, 48);
            myNotebook.setContentAreaFilled(false);
            myNotebook.setBorderPainted(false);
            myNotebook.addMouseListener(mouseOption);
            myNotebook.setText("notebook");
            desktop.add(myNotebook, Integer.MIN_VALUE + 1);
            
            
            JButton myPanel= new JButton();
            myPanel.setIcon(new ImageIcon(this.getClass().getResource("/lib/paper_plane.png")));
            myPanel.setBounds(20, 156, 48, 48);
            myPanel.setContentAreaFilled(false);
            myPanel.setBorderPainted(false);
            myPanel.addMouseListener(mouseOption);
            myPanel.setText("panel");
            desktop.add(myPanel, Integer.MIN_VALUE + 1);
            
            JButton mySunModel = new JButton();
            mySunModel.setIcon(new ImageIcon(this.getClass().getResource("/lib/earth.net.png")));
            mySunModel.setBounds(20, 224, 48, 48);
            mySunModel.setContentAreaFilled(false);
            mySunModel.setBorderPainted(false);
            mySunModel.addMouseListener(mouseOption);
            mySunModel.setText("sunModel");
            desktop.add(mySunModel, Integer.MIN_VALUE + 1);
            
                    
            desktop.add(backgroundImage, new Integer(Integer.MIN_VALUE));
            
            
            this.getContentPane().add(desktop, BorderLayout.CENTER);
            this.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
            this.setVisible(true);
            
        }
        
        private class MouseOption extends MouseAdapter {
            private int count;
            private Timer timer = new Timer();
            private String str = null;
            
            private class MyTimerTask extends TimerTask {
                JButton button = null;
                
                
                public MyTimerTask(JButton button) {
                    this.button = button;
                    
                }
                
                
    
    
                @Override
                public void run() {
                    //超过0.4s且点击了一次
                    if(count == 1) {
                        count = 0;
                    }
                    //在0.4s内点击两次
                    if(count == 2) {
                        //JDK7.0以上支持switch字符串
                        switch(str) {
                        case "fileChooser" : 
                            JFileChooser fileChooser = new JFileChooser();
                            fileChooser.setFileSelectionMode(JFileChooser.FILES_AND_DIRECTORIES);
                            fileChooser.showOpenDialog(Desktop.this);
                            File file = fileChooser.getSelectedFile();
                            if(file != null) {
                                
                                System.out.println(file.getAbsolutePath());
                                
                            }
                            break;
                        case "notebook" : 
                            //调用windows系统自带的notepad
                            /*try {
                                Runtime.getRuntime().exec("notepad");
                            } catch (IOException e) {
                                e.printStackTrace();
                            }*/
                            //调用自个写的一个特简易的记事本程序
                            Notepad notepad = new Notepad();
                            desktop.add(notepad);
                            notepad.toFront();
                            
                            break;
                        case "compute" :
                            //打开windows的文件管理器
                            try {
                                java.awt.Desktop.getDesktop().open(new File(System.getProperty("user.home")));
                            } catch (IOException e) {
                                e.printStackTrace();
                            }
                            break;
                        case "panel" : 
                            //启动飞机大战游戏
                            new PanelGame();
                            break;
                            
                        case "sunModel" :
                            //启动太阳系模型，虽然没啥用，用来装B
                            new SunModel();
                            break;
                        }
                        
                        button.setContentAreaFilled(false);
                        count = 0;
                    }
                    
                }
                
                
            }
            
            /**
             * 添加鼠标点击事件
             */
            @Override
            public void mouseClicked(MouseEvent e) {
                JButton button = (JButton) e.getSource();
                button.setContentAreaFilled(true);
                str = button.getText();
                count ++;//用于记录点击次数
                //定制双击事件，使用定时器，每次点击后，延时0.4
                timer.schedule(new MyTimerTask(button), 400);
                
            }
            
            
            
            
            
        }
        
        public static void main(String[] args) {
            new Desktop("桌面");
        }
        
    
    }

三、Notepad简易小程序

1、效果图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/525f9c5ca3edd15f4120bcb38ba2d589.png)

2、Notepad.java的代码

    package com.swing;
    
    import java.awt.Desktop;
    import java.awt.Font;
    import java.awt.event.ActionEvent;
    import java.awt.event.ActionListener;
    import java.awt.event.KeyAdapter;
    import java.awt.event.KeyEvent;
    import java.awt.event.MouseAdapter;
    import java.awt.event.MouseEvent;
    import java.io.BufferedReader;
    import java.io.BufferedWriter;
    import java.io.File;
    import java.io.FileInputStream;
    import java.io.FileOutputStream;
    import java.io.IOException;
    import java.io.InputStreamReader;
    import java.io.OutputStreamWriter;
    
    import javax.swing.JFileChooser;
    import javax.swing.JFrame;
    import javax.swing.JInternalFrame;
    import javax.swing.JMenu;
    import javax.swing.JMenuBar;
    import javax.swing.JMenuItem;
    import javax.swing.JOptionPane;
    import javax.swing.JScrollPane;
    import javax.swing.JTextArea;
    
    /**
     * 简易笔记本
     * @author may
     *
     */
    public class Notepad extends JInternalFrame {
    
        private static final long serialVersionUID = -6148113299360403243L;
    
        private JMenuBar menuBar = null;//菜单栏
        private JTextArea textArea = null;//输入框
        private JScrollPane scrollPane = null;//带滚动条的面板
        private MyAction myAction = new MyAction();//事件对象
        private String dir = null;//保存打开过或者保存过文件的文件夹
        private String fileDir = null;//保存你打开文件的文件夹
        private boolean ctrlClick = false;//用于检测当前，你是否按下了Ctrl键
        private boolean sClick = false;//用于检测当前，你是否按下了s键
    
        public Notepad() {
            super("notepad");
            this.setSize(600, 500);//窗口的大小
            menuBar = new JMenuBar();//创建菜单栏
            JMenu menu1 = new JMenu("文件");//创建菜单
            JMenuItem menuItem2 = new JMenuItem("打开");//创建菜单项
            JMenuItem menuItem4 = new JMenuItem("保存");//创建菜单项
            menuItem4.addActionListener(myAction);//绑定事件
            menuItem2.addActionListener(myAction);//绑定事件
            JMenuItem menuItem3 = new JMenuItem("打开文件所在目录");
            menuItem3.addActionListener(myAction);
            menu1.add(menuItem2);
            menu1.add(menuItem3);
            menu1.add(menuItem4);
    
            JMenu menu2 = new JMenu("版本信息");
            menu2.addMouseListener(new MouseAdapter() {
    
                @Override
                public void mouseClicked(MouseEvent e) {
                    //定义弹窗后的按钮的文字
                    String[] options = {"确定"};
                    //创建一个弹窗
                    JOptionPane.showOptionDialog(Notepad.this, "version:0.1-snapshoot", "关于", JOptionPane.OK_OPTION, JOptionPane.INFORMATION_MESSAGE, null, options, "确定");
                }
                
                
            });
            menuBar.add(menu1);
            menuBar.add(menu2);
            this.setJMenuBar(menuBar);
            textArea = new JTextArea();
            //添加键盘检测事件
            textArea.addKeyListener(new keyOption());
            // this.getContentPane().add(menuBar, BorderLayout.NORTH);
            //设置字体
            textArea.setFont(new Font("微软雅黑", Font.PLAIN, 18));
            scrollPane = new JScrollPane(textArea);
            //当文本水平溢出时出现滚动条
            scrollPane.setHorizontalScrollBarPolicy(JScrollPane.HORIZONTAL_SCROLLBAR_AS_NEEDED);
            //当文本垂直溢出时出现滚动条
            scrollPane.setVerticalScrollBarPolicy(JScrollPane.VERTICAL_SCROLLBAR_AS_NEEDED);
            this.getContentPane().add(scrollPane);
            //居中显示
            //this.setLocationRelativeTo(null);
            //最小化
            this.setIconifiable(true);
            //可关闭
            this.setClosable(true);
            //可改变大小
            this.setResizable(true);
            //销毁窗口
            this.setDefaultCloseOperation(JFrame.DISPOSE_ON_CLOSE);
            this.setVisible(true);
    
        }
    
        /**
         * 打开文件选择和保存对话框
         * @param flag
         */
        public void openChooseDialog(String flag) {
            BufferedReader reader = null;
            JFileChooser fileChooser = new JFileChooser();
            if(dir != null) {
                //定位上次打开和保存过文件的位置
                fileChooser.setCurrentDirectory(new File(dir));
            }
            
                switch(flag) {
                case "打开" :
                    //指定它的父窗口
                    fileChooser.showOpenDialog(Notepad.this);
                    //定义文件选择模式
                    fileChooser.setFileSelectionMode(JFileChooser.FILES_ONLY);
                    
                    File file = fileChooser.getSelectedFile();
                    if(file != null) {
                        try {
                            //得到选择文件的路径
                            dir = file.getAbsolutePath();
                            fileDir = dir;
                            dir = dir.substring(0, dir.lastIndexOf("\\") + 1);
                            reader = new BufferedReader(new InputStreamReader(new FileInputStream(file), "utf-8"));
                            String str = reader.readLine();
                            textArea.setText("");
                            //读取文件内容
                            while(str != null) {
                                textArea.append(str + "\n");
                                str = reader.readLine();
                                
                            }
                        } catch (Exception ex) {
                            ex.printStackTrace();
                        } finally {
                            if(reader != null) {
                                try {
                                    reader.close();
                                } catch (IOException e) {
                                    e.printStackTrace();
                                }
                            }
                            
                        }
                    }
                    
                    break;
                case "保存" :
                    //打开保存文件的对话框
                    fileChooser.showSaveDialog(Notepad.this);
                    //得到保存文件后的文件对象
                    File saveFile = fileChooser.getSelectedFile();
                    if(saveFile != null) {
                        //得到保存文件的路径
                        String absolutePath = saveFile.getAbsolutePath();
                        fileDir = absolutePath;
                        FileOutputStream out = null;
                        BufferedWriter buffOut = null;
                        
                        dir = absolutePath.substring(0, absolutePath.lastIndexOf("\\") + 1);
                        //保存文件
                        try {
                            out = new FileOutputStream(absolutePath);
                            buffOut = new BufferedWriter(new OutputStreamWriter(out));
                            String text = textArea.getText();
                            if(text != null) {
                                buffOut.write(text);
                                
                            }
                            buffOut.flush();
                        } catch (Exception e) {
                            e.printStackTrace();
                        } finally {
                            try {
                                if(out != null)  {
                                    out.close();
                                    
                                }
                                if(buffOut != null) {
                                    buffOut.close();
                                }
                            } catch (IOException e1) {
                                e1.printStackTrace();
                            }
                        }
                        
                    }
                    
                    break;
                case "打开文件所在目录":
                    if(dir != null) {
                        try {
                            //打开文件目录
                            Desktop.getDesktop().open(new File(dir));
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }
                    break;
                }
    
        }
        
        /**
         * 事件监听类
         * @author may
         *
         */
        private class MyAction implements ActionListener {
            
    
            @Override
            public void actionPerformed(ActionEvent e) {
                JMenuItem item = (JMenuItem) e.getSource();
                String flag = item.getText();
                switch (flag) {
                case "打开":
                    openChooseDialog(flag);
                    break;
                case "保存":
                    openChooseDialog(flag);
                    break;
                case "打开文件所在目录":
                    openChooseDialog(flag);
                    break;
                }
    
            }
    
        }
        /**
         * 键盘监听内部类
         * @author may
         *
         */
        private class keyOption extends KeyAdapter {
            
    
            @Override
            public void keyPressed(KeyEvent e) {
                int keyCode = e.getKeyCode();
                if(17 == keyCode) {
                    ctrlClick = true;
                } else if(83 == keyCode) {
                    sClick = true;
                    
                }
                //判断Ctrl与s键是否按下，按下就开始保存
                if(ctrlClick && sClick) {
                    FileOutputStream out = null;
                    BufferedWriter buffOut = null;
                    
                    try {
                        if(fileDir != null) {
                            out = new FileOutputStream(fileDir);
                            buffOut = new BufferedWriter(new OutputStreamWriter(out));
                            String text = textArea.getText();
                            if(text != null) {
                                buffOut.write(text);
                                
                                
                            }
                            buffOut.flush();
                        } else {
                            openChooseDialog("保存");
                            
                        }
                    } catch (Exception ex) {
                            ex.printStackTrace();
                    } finally {
                        try {
                            if(out != null)  {
                                out.close();
                                
                            }
                            if(buffOut != null) {
                                buffOut.close();
                            }
                        } catch (IOException e1) {
                            e1.printStackTrace();
                        }
                    }
                        
                        
                }
            }
    
            @Override
            public void keyReleased(KeyEvent e) {
                int keyCode = e.getKeyCode();
                if(17 == keyCode) {
                    ctrlClick = false;
                } else if(83 == keyCode) {
                    sClick = false;
                    
                }
            }
            
            
            
            
        } 
        
    
    }

四、飞机大战简易小游戏

1、效果图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/6e4d3837dc585408972e2d2aa8be577a.png)



2、代码

（1）主类PanelGame.java

    package com.swing.plane;
    
    import java.awt.Color;
    import java.awt.Font;
    import java.awt.Frame;
    import java.awt.Graphics;
    import java.awt.Image;
    import java.awt.event.KeyAdapter;
    import java.awt.event.KeyEvent;
    import java.awt.event.WindowAdapter;
    import java.awt.event.WindowEvent;
    import java.awt.image.BufferedImage;
    import java.util.ArrayList;
    import java.util.List;
    import java.util.Random;
    
    import com.swing.util.ImageLoadUtil;
    
    /**
     * 主类
     * @author may
     *
     */
    public class PanelGame extends Frame {
    
        private static final long serialVersionUID = 6719627071124599012L;
        //我方飞机场
        private List<Panel> Goodpanels = new ArrayList<Panel>();
        //敌军飞机场
        private List<Panel> panels = new ArrayList<Panel>();
        //公共的子弹工厂
        private List<Shell> shells = new ArrayList<Shell>(); 
        //随机
        private Random random = new Random();
        //战场背景
        private static Image image = ImageLoadUtil.loadImage("/lib/bg.jpg");
        //缓冲图
        private BufferedImage buffImage = new BufferedImage(500, 500, BufferedImage.TYPE_INT_RGB);    
        //爆炸
        private List<Explode> explodes = new ArrayList<Explode>();
        //杀敌数
        public int killEnemyCount = 0;
        //死亡次数
        public int deadCount = 0;
        
        public PanelGame() {
            this.setSize(500,500);
            this.setLocation(300, 100);
            this.addWindowListener(new WindowAdapter() {
                
                @Override
                public void windowClosing(WindowEvent e) {
                    PanelGame.this.dispose();
                }
                
            });
            //在显示（画窗口前new出来，否则会报空指针异常）
            //panel = new Panel(100,100, false);
            this.addKeyListener(new keyCtrl());
            this.createGoodPanels(1);
            this.setVisible(true);
            new Thread(new MyThread()).start();
            
        }
        
        
        
        public void createPanels(int num) {
            for(int i = 0; i < num; i++) {
                panels.add(new Panel(this, true));
            }
            
        }
        
        public void createGoodPanels(int num) {
            if(Goodpanels.size() <= 0) {
                
                for(int i = 0; i < num; i++) {
                    Goodpanels.add(new Panel(452, 250, false, this));
                }
                
            }
        }
        
        
        
        public List<Explode> getExplodes() {
            return explodes;
        }
    
    
    
        public List<Shell> getShells() {
            return shells;
        }
    
    
    
        public List<Panel> getPanels() {
            return panels;
        }
        
        
        
    
        public List<Panel> getGoodpanels() {
            return Goodpanels;
        }
    
    
    
        @Override
        public void paint(Graphics g) {
            g.drawImage(buffImage, 0, 0, null);
        }
        
        @Override
        public void update(Graphics g) {
            Graphics warG = buffImage.getGraphics();
            warG.fillRect(0, 0, 500, 500);
            warG.drawImage(image, 0, 0, 500, 500, null);
            Color c = warG.getColor();
            warG.setColor(Color.gray);
            warG.setFont(new Font("微软雅黑", Font.PLAIN, 18));
            warG.drawString("击毁敌机：" + killEnemyCount, 10, 50);
            warG.drawString("死亡次数：" + deadCount, 10, 80);
            
            for(int i = 0; i < Goodpanels.size(); i++) {
                Goodpanels.get(i).draw(warG);
                warG.drawString("生命值：" + Goodpanels.get(i).lifeValue, 10, (i+1)*30 + 80);
            }
            warG.setColor(c);
            for(int i = 0; i < panels.size(); i++) {
                panels.get(i).draw(warG);
            }
            for(int i = 0; i < shells.size(); i++) {
                shells.get(i).draw(warG);
            }
            for(int i = 0; i < explodes.size(); i++) {
                explodes.get(i).draw(warG);
            }
            paint(g);
            warG.setColor(c);
            if(this.getPanels().size() < 3) {
                this.createPanels(random.nextInt(6) + 1);
                
            }
        }
        
        
        
        
        public List<Panel> getPanel() {
            return Goodpanels;
        }
    
    
    
        public static void main(String[] args) {
            new PanelGame();
        }
        
        private class keyCtrl extends KeyAdapter {
    
            @Override
            public void keyPressed(KeyEvent e) {
                int keyCode = e.getKeyCode();
                if(keyCode == KeyEvent.VK_F1) {
                    createGoodPanels(1);
                    
                }
                
                for(int i = 0; i < Goodpanels.size(); i++) {
                    
                    Goodpanels.get(i).keyPressed(e);
                }
                
            }
    
            @Override
            public void keyReleased(KeyEvent e) {
                for(int i = 0; i < Goodpanels.size(); i++) {
                    
                    Goodpanels.get(i).keyReleased(e);
                }
            }
            
            
            
        }
        
        private class MyThread implements Runnable {
    
            @Override
            public void run() {
                while(true) {
                    repaint();
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                
            }
            
        }
        
    }

(2)Panel.java

    package com.swing.plane;
    
    import java.awt.Color;
    import java.awt.Graphics;
    import java.awt.Image;
    import java.awt.Rectangle;
    import java.awt.event.KeyEvent;
    import java.util.List;
    import java.util.Random;
    
    import com.swing.util.ImageLoadUtil;
    /**
     * 飞机类
     * @author may
     *
     */
    public class Panel {
        
        private int x;//位置
        private int y;//位置
        private static final int WIDTH = 48;//飞机的大小
        private static final int HEIGHT = 48;//飞机的大小
        private static final int WAR_WIDTH = 500;//游戏窗口的大小
        private static final int WAR_HEIGHT = 500;//游戏窗口的大小
        private boolean up = false, right = false, down = false, left = false;//飞机的移动方向
        private boolean enemy = true;//默认为敌机
        private static Image[] images = new Image[2];//敌机与我方的飞机图片
        private List<Shell> shells = null;//子弹容器
        private int step = 1;//移动的步数
        private boolean isLive = true;//默认飞机生存
        private int oldX = x;//记录飞机上一次的位置
        private int oldY = y;//记录飞机上一次的位置
        private int keep = 0;//敌机自动发射子弹的间隔
        private Random random = new Random();//定义随机数，用于定义敌机的子弹发射的时间间隔
        private Director dir = Director.STOP;//飞机默认是不动的
        private PanelGame game;//游戏主类
        public int lifeValue = 90;//飞机的生命值
        
        static {
            for(int i = 0; i < images.length; i++) {
                images[i] = ImageLoadUtil.loadImage("/lib/plane" + (i + 1) + ".png");
                images[i].getWidth(null);
                
            }
            
        }
        
        
        public Panel(PanelGame game, boolean enemy) {
            this.game = game;
            this.enemy = enemy;
            //如果是敌机，定义它初始位置
            if(this.enemy) {
                this.x = - random.nextInt(3 * WIDTH) ;
                this.y = random.nextInt(WAR_HEIGHT - 2 * HEIGHT) + HEIGHT;
            }
            this.shells = game.getShells();
        }    
        
        public Panel(int x, int y, boolean enemy, PanelGame game) {
            this.x = x;
            this.y = y;
            this.enemy = enemy;
            this.game = game;
            this.shells = game.getShells();
        }
        
        
        
        public void draw(Graphics g) {
            //如果飞机还活着，就画它
            if(this.isLive) {
                
                Color c = g.getColor();
                if(enemy) {
                    g.drawImage(images[1], x, y, WIDTH, HEIGHT, null);
                    
                } else {
                    g.drawImage(images[0], x, y, WIDTH, HEIGHT, null);
                }
                g.setColor(c);
                g.setColor(c);
                
                this.move();
                if(enemy) {
                    this.hit(game);
                }
            }
            
            
        }
        
        /**
         * 创建子弹
         */
        public void createShells() {
            if(!enemy) {
                this.getShells().add(new Shell(x - 10, y + 20, this, false, game));
                
            } else {
                
                this.getShells().add(new Shell(x + 50, y + 20, this, true, game));
            }
        
        }
        
        
    
        public boolean isEnemy() {
            return enemy;
        }
    
        public int getX() {
            return x;
        }
        
        
    
        public List<Shell> getShells() {
            //shells = game.getShells();
            return shells;
        }
        //如果飞机被击毁，飞机在飞机战场上消失（主类的容器）
        public void removedeadPlane() {
            this.isLive = false;
            if(!this.isLive && this.enemy) {
                game.getPanels().remove(this);
            } 
            if(!this.isLive && !this.enemy) {
                game.getPanel().remove(this);
            } 
            
        }
    
        public boolean isLive() {
            return isLive;
        }
        
        public void setLive(boolean isLive) {
            this.isLive = isLive;
        }
    
        public int getY() {
            return y;
        }
        //确定方向，这些方向的确定是通过按键监听来确定
        private void directer() {
            if( left && !down && !right && !up ) {
                dir = Director.L;
            }
            
            else if( left && up && !right && !down ) {
                dir = Director.LU;    
            }
            
            else if( up && !left && !right && !down ) {
                dir = Director.U;
            }
            
            else if( up && right && !left && !down ) {
                dir = Director.RU;
            }
            
            else if( right && !up && !left && !down ) {
                dir = Director.R;
            }
            
            else if( right && down && !up && !left ) {
                dir = Director.RD;
            }
            
            else if( down && !up && !right && !left ) {
                dir = Director.D;
            }
            
            else if( left && down && !right && !up ) {
                dir = Director.LD;
            }
            
            else if( !left && !up && !right && !down ) {
                dir = Director.STOP;
            }
        }
    
        //根据方向的移动方法
        public void move() {
            oldX = x;
            oldY = y;
            
            switch(dir) {
            case L:
                x -= step;
                break;
            case LU:
                x -= step;
                y -= step;
                break;
            case U:
                y -= step;
                break;
            case RU:
                x += step;
                y -= step;
                break;
            case R:
                x += step;
                break;
            case RD:
                x += step;
                y += step;
                break;
            case D:
                y += step;
                break;
            case LD:
                x -= step;
                y += step;
                break;
            case STOP:
                break;
            }
            //如果不是敌机，不允许它跑出战场
            if(!enemy) {
                
                if(x > (WAR_WIDTH - 48) || x < 0) {
                    x = oldX;
                    
                }
                if(y > (WAR_HEIGHT - 48) || y < 30) {
                    y = oldY;
                    
                }
                
            }
            //如果是敌机，确定它的方向
            if(enemy) {
                dir = Director.R;
                //每隔50个刷新周期，发射一枚子弹
                if(keep == 0) {
                    keep = 50;
                    this.createShells();
                }
                keep --;
                //如果敌机逃出战场，摧毁它
                if(x > WAR_WIDTH) {
                    game.getPanels().remove(this);
                    
                }
                
            }
            
        }
        
        //键盘按下监听事件，由主类调用
        public void keyPressed(KeyEvent e) {
            int keyCode = e.getKeyCode();
            switch(keyCode) {
            case KeyEvent.VK_UP  : 
                this.up = true;
                break;
            case KeyEvent.VK_RIGHT :
                this.right = true;
                break;
            case KeyEvent.VK_DOWN :
                this.down = true;
                break;
            case KeyEvent.VK_LEFT  :
                this.left = true;
                break;
            }
            this.directer();
            
        }
    
        //键盘抬起监听事件，由主类调用
        public void keyReleased(KeyEvent e) {
            int keyCode = e.getKeyCode();
            switch(keyCode) {
            case KeyEvent.VK_UP : 
                this.up = false;
                break;
            case KeyEvent.VK_RIGHT :
                this.right = false;
                break;
            case KeyEvent.VK_DOWN :
                this.down = false;
                break;
            case KeyEvent.VK_LEFT :
                this.left = false;
                break;
            case KeyEvent.VK_CONTROL :
                this.createShells();
                break;
            }
            this.directer();
        }
        //矩形碰撞检测
        public Rectangle getRectangle() {
            
            return new Rectangle(this.x, this.y, WIDTH, HEIGHT);
            
        }
        
        //撞到了就爆炸
            public void createExplode() {
                
                game.getExplodes().add(new Explode(this.x, this.y, game));
            }
        
        public void hit(PanelGame game) {
            
            if(this.enemy) {
                List<Panel> p = game.getGoodpanels();
                for(int i = 0; i < p.size(); i++) {
                    if(this.getRectangle().intersects(p.get(i).getRectangle())) {
                        game.getPanels().remove(this);
                        
                        p.get(i).lifeValue -= 30;
                            
                        
                        if(p.get(i).lifeValue <= 0) {
                            
                            game.getGoodpanels().remove(p.get(i));
                            
                        } 
                        this.createExplode();
                        
                        
                    }
                }
                
            } 
            
        }
    
    }

(3)、Shell.java炮弹类

    package com.swing.plane;
    
    import java.awt.Color;
    import java.awt.Graphics;
    import java.awt.Rectangle;
    import java.util.List;
    /**
     * 炮弹类
     * @author may
     *
     */
    public class Shell {
        
        private int x;
        private int y;
        private static final int WIDTH = 10;
        private static final int HEIGHT = 10;
        private boolean enemy = false;
        private boolean isLive = true;
        private Panel panel;
        private PanelGame game;
        private int step =2;
        
        
        public Shell(int x, int y, Panel panel, boolean enemy, PanelGame game) {
            this.x = x;
            this.y = y;
            this.panel = panel;
            this.enemy = enemy;
            this.game = game;
        }
        
        public void draw(Graphics g) {
            
            if(this.isLive) {
                
                Color c = g.getColor();
                if(enemy) {
                    
                    g.setColor(Color.red);
                } else {
                    
                    g.setColor(Color.yellow);
                }
                g.fillOval(this.x, this.y, WIDTH, HEIGHT);
                g.setColor(c);
                move();
                this.hit(game);
            }
            
        }
        
        public Rectangle getRectangle() {
            Rectangle rectangle = new Rectangle(x, y, WIDTH, HEIGHT);
            
            return rectangle;
        }
        
        
        
        public boolean isLive() {
            return isLive;
        }
    
        public void setLive(boolean isLive) {
            this.isLive = isLive;
        }
    
        public void move() {
            if(!enemy) {
                x -= step;
            } else {
                x += step;
            }
            
            if(x < 10 || x > 500) {
                this.setLive(false);
                panel.getShells().remove(this);
            }
            
        }
    
        public void setX(int x) {
            this.x = x;
        }
    
        public void setY(int y) {
            this.y = y;
        }
        //撞到了就爆炸
        public void createExplode() {
            
            game.getExplodes().add(new Explode(this.x, this.y, game));
        }
        //碰撞检测方法
        public void hit(PanelGame game) {
            List<Panel> enemys = game.getPanels();
            List<Panel> goods = game.getPanel();
            for(int i = 0; i < enemys.size(); i++) {
                if(this.enemy != enemys.get(i).isEnemy()) {
                    
                    if(this.getRectangle().intersects(enemys.get(i).getRectangle())) {
                        this.setLive(false);
                        panel.getShells().remove(this);
                        enemys.get(i).removedeadPlane();
                        this.createExplode();
                        game.killEnemyCount ++;
                    }
                }
                
            }
            
            for(int i = 0; i < goods.size(); i++) {
                if(this.enemy != goods.get(i).isEnemy()) {
                    
                    if(this.getRectangle().intersects(goods.get(i).getRectangle())) {
                        this.setLive(false);
                        panel.getShells().remove(this);
                        goods.get(i).lifeValue -= 30;
                        
                        if(goods.get(i).lifeValue <= 0) {
                            game.deadCount ++;
                            goods.get(i).removedeadPlane();
                            
                        } 
                        this.createExplode();
                    }
                }
                
            }
            
        }
        
        
    
    }

（4）爆炸类Explode.java

    package com.swing.plane;
    
    import java.awt.Color;
    import java.awt.Graphics;
    import java.awt.Image;
    
    import com.swing.util.ImageLoadUtil;
    /**
     * 爆炸类
     * @author may
     *
     */
    public class Explode {
        
        private int x;//爆炸位置
        private int y;//爆炸位置
        private static Image[] images = new Image[10];
        private int count = 0;//当前播放到的图片在数组中的下标
        private PanelGame game;//持有主窗口的引用
        
        //静态加载图片
        static {
            for(int i = 0; i < images.length; i++) {
                images[i] = ImageLoadUtil.loadImage("/lib/explode/" + (i) + ".gif");
                //避免Java的懒加载，让它一用到就可直接用，否则可能一开始的一两张图将看不到
                images[i].getHeight(null);
            }
            
        }
        
        /**
         * 
         * @param x 位置
         * @param y 位置
         * @param game 游戏主类
         */
        public Explode(int x, int y, PanelGame game) {
            this.x = x;
            this.y = y;
            this.game = game;
        }
        
        public void draw(Graphics g) {
            Color c = g.getColor();
            if(count == 3) {
                //播放完后将自己从主类容器中去掉
                game.getExplodes().remove(this);
            }
            g.drawImage(images[count ++], this.x - images[count ++].getWidth(null)/2 , this.y - images[count ++].getHeight(null)/2, 30, 30,  null);
            g.setColor(c);
        }
    
    }

（5）方向枚举常量

    package com.swing.plane;
    
    /**
     * 定义飞机运动的方向
     * @author may
     *
     */
    public enum Director {
        
        U, RU, R, RD, D , LD, L, LU, STOP;
        
        
        
    }

（6）图片加载辅助类

    package com.swing.util;
    
    import java.awt.Image;
    import java.awt.image.BufferedImage;
    import java.io.IOException;
    
    import javax.imageio.ImageIO;
    
    public class ImageLoadUtil {
    
        private ImageLoadUtil() {}
        
        public static Image loadImage(String path) {
            
            BufferedImage image = null;
            
            try {
                image = ImageIO.read(ImageLoadUtil.class.getResource(path));
            } catch (IOException e) {
                e.printStackTrace();
            }
    
            
            return image;
            
        }
        
    }

五、太阳系模型（这个是本人以前刚学java的时候看的那个尚学堂java300讲视频里的,感觉好玩就自己弄了一个）

1、效果

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/a0fdfad0c0660ae88da368ac57f83df7.png)

2、代码

（1）定义图形大小的一些常量类

    package com.swing.sunModel;
    
    /**
     * 常量类
     * @author may
     *
     */
    public class Constant {
        
        public static final int GAME_WIDTH = 800;
        public static final int GAME_HEIGHT = 600;
        
        public static final int FIXED_WIDTH = 40;
        public static final int FIXED_HEIGHT = 35;
    
    }

（2）Star.java接口类

    package com.swing.sunModel;
    
    import java.awt.Graphics;
    
    /**
     * 行星的抽象类
     * @author may
     *
     */
    public interface Star {
        
        public int getX();
        
        public int getY();
        
        public int getWidth();
        
        public int getHeight();
        
        public void draw(Graphics g);
    
    }

（3）FixedStar.java恒星类

    package com.swing.sunModel;
    
    import java.awt.Color;
    import java.awt.Graphics;
    import java.awt.Image;
    import java.util.ArrayList;
    import java.util.List;
    
    public class FixedStar implements Star {
    
        private int x;//x轴位置
        private int y;//y轴位置
        private int fixed_width;//物体的宽度
        private int fixed_height;//物体的高度
        private List<Star> planets = new ArrayList<Star>();//用于存储行星的容器
        private static Image image = ImageLoadUtil.loadImage("/lib/Sun.png");////恒星的图片
        
        
        public FixedStar(int fixed_width, int fixed_height) {
            //计算出xy的值，是它居中显示
            this.x = (Constant.GAME_WIDTH - fixed_width)/2;
            this.y = (Constant.GAME_HEIGHT - fixed_height)/2;
            this.fixed_width = fixed_width;
            this.fixed_height = fixed_height;
            this.createPlanet();
            
            
        }
        /**
         * 创建围绕它转的所有行星
         */
        private void createPlanet() {
            Star earth= new Planet(100, 50, 30, 30, this, "/lib/earth.net.png", 0.09);
            planets.add(earth);
            planets.add(new Planet(20, 15, 15, 15, earth, "/lib/venus.png", 0.3, true));
            planets.add(new Planet(200, 100, 30, 30, this, "/lib/goog_mars.png", 0.08));
            planets.add(new Planet(250, 150, 30, 30, this, "/lib/uranus.png", 0.05));
            planets.add(new Planet(350, 200, 30, 30, this, "/lib/venus.png", 0.03));
        }
        
        /**
         * 绘画方法
         * @param g
         */
        public void draw(Graphics g) {
            Color c = g.getColor();
            g.setColor(Color.red);
            //g.fillOval(this.x, this.y, fixed_width, fixed_height);
            g.drawImage(image, this.x, this.y, fixed_width, fixed_height, null);
            g.setColor(c);
            //画出每个行星
            for(int i = 0; i < this.planets.size(); i++) {
                planets.get(i).draw(g);
                
            }
        }
        
        public int getX() {
            return x;
        }
        public void setX(int x) {
            this.x = x;
        }
        public int getY() {
            return y;
        }
        public void setY(int y) {
            this.y = y;
        }
        public void setFixed_width(int fixed_width) {
            this.fixed_width = fixed_width;
        }
        public void setFixed_height(int fixed_height) {
            this.fixed_height = fixed_height;
        }
        @Override
        public int getWidth() {
            return fixed_height;
        }
        @Override
        public int getHeight() {
            return fixed_height;
        }
        
        
        
        
    }

（4）父窗口类MyFrame.java

    package com.swing.sunModel;
    
    import java.awt.Frame;
    import java.awt.event.WindowAdapter;
    import java.awt.event.WindowEvent;
    
    /**
     * 定义一个父类，将常用的代码写到此处
     * @author may
     *
     */
    public class MyFrame extends Frame {
        
        private static final long serialVersionUID = 8133309356114432110L;
    
        public MyFrame() {
            this.setSize(Constant.GAME_WIDTH, Constant.GAME_HEIGHT);
            this.setLocationRelativeTo(null);
            
            
            this.addWindowListener(new WindowAdapter() {
            
                @Override
                public void windowClosing(WindowEvent e) {
                    MyFrame.this.dispose();
                }
            });
            this.setVisible(true);
            
        }
        
    
    }

（5）行星类Planet.java

    package com.swing.sunModel;
    
    import java.awt.Color;
    import java.awt.Graphics;
    import java.awt.Image;
    import java.util.ArrayList;
    import java.util.List;
    
    public class Planet implements Star {
    
        private int x;//行星的位置
        private int y;//行星的位置
        private int planet_width;//行星的大小
        private int planet_height;//行星的大小
        private int longXpis; //离恒星的距离，就是椭圆的长轴
        private int shortXpis;//离恒星的距离，就是椭圆的短轴
        private Star fixedStar;//恒星
        private double degree = 1;//角度
        private Image image = null;//行星的图片，由于每个行星不同，所以定义为非静态的
        private double speed = 0.01;//默认的改变角度的速度
        private List<Planet> moon = new ArrayList<Planet>();//定义行星的子行星，如地球的月亮
        private boolean satellite = false;
    
        public Planet(int x, int y) {
            this.x = x;
            this.y = y;
    
        }
        /**
         * 
         * @param longXpis 长轴
         * @param shortXpis 短轴
         * @param planet_width 行星的宽度
         * @param planet_height 行星的高度
         * @param fixedStar 中心星体，即恒星
         * @param path 图片的位置
         * @param speed 角度改变的速度
         */
        public Planet(int longXpis, int shortXpis, int planet_width, int planet_height, Star fixedStar, String path,
                double speed) {
            //定义离中心星体的初始距离
            this(fixedStar.getX() + fixedStar.getWidth() / 2 + longXpis - planet_width / 2,
                    fixedStar.getY() + fixedStar.getHeight() / 2 - planet_height / 2);
            this.longXpis = longXpis;
            this.shortXpis = shortXpis;
            this.planet_height = planet_height;
            this.planet_width = planet_width;
            this.fixedStar = fixedStar;
            this.image = ImageLoadUtil.loadImage(path);
            this.speed = speed;
        }
        
        public Planet(int longXpis, int shortXpis, int planet_width, int planet_height, Star planetStar, String path,
                double speed, boolean satellite) {
            this(longXpis, shortXpis, planet_width, planet_height, planetStar, path,
                    speed);
            this.satellite = satellite;
        }
        
        /**
         * 绘画方法
         * @param g
         */
        public void draw(Graphics g) {
            Color c = g.getColor();
            g.setColor(Color.gray);
            //画出行星图片
            g.drawImage(image, this.x, this.y, planet_width, planet_height, null);
            //画出行星的运行轨迹
            if(!satellite) {
                g.drawOval(fixedStar.getX() - (longXpis - this.planet_width / 2),
                        fixedStar.getY() - (shortXpis - this.planet_height / 2), this.longXpis * 2, this.shortXpis * 2);
                
            }
            g.setColor(c);
            //以下的代码使行星做椭圆运动
            this.x = (int) (fixedStar.getX() + longXpis * Math.cos(degree));
            this.y = (int) (fixedStar.getY() + shortXpis * Math.sin(degree));
            if(degree < 2 * Math.PI) {
                degree += speed;
                
            } else {
                degree = 0;
            }
            //画子行星
            for(int i = 0; i < moon.size(); i++) {
                moon.get(i).draw(g);
            }
        }
    
    
    
        public int getX() {
            return x;
        }
    
        public int getY() {
            return y;
        }
        @Override
        public int getWidth() {
            return planet_width;
        }
        @Override
        public int getHeight() {
            return planet_height;
        }
        
        
    
    }

(6)主类sunMode.java

    package com.swing.sunModel;
    
    import java.awt.Graphics;
    import java.awt.Image;
    import java.awt.image.BufferedImage;
    
    public class SunModel extends MyFrame {
        
        private static final long serialVersionUID = -5330545593499050676L;
        private static Image image = ImageLoadUtil.loadImage("/lib/star.jpg");
        //恒星对象
        private FixedStar fixedStar = new FixedStar(Constant.FIXED_WIDTH, Constant.FIXED_HEIGHT);
        //创建一个缓冲图，用于双缓冲，避免闪烁出现
        private BufferedImage buffImage = new BufferedImage(Constant.GAME_WIDTH, Constant.GAME_HEIGHT, BufferedImage.TYPE_INT_RGB);
        
        public SunModel() {
            super();
            this.setResizable(false);
            new Thread(new MyThread()).start();
            
        }
        
        @Override
        public void paint(Graphics g) {
            //将缓冲图直接画上去
            g.drawImage(buffImage, 0, 0, null);
        }
        
        //重写update方法，进行双缓冲，将需要显示的物体全画到缓冲图上，然后一次性显示出来
        @Override
        public void update(Graphics g) {
            Graphics sunG = buffImage.getGraphics();
            //将背景图画上
            sunG.drawImage(image, 0, 0, null);
            fixedStar.draw(sunG);
            paint(g);
        }
    
        public static void main(String[] args) {
            new SunModel();
        }
        
        private class MyThread implements Runnable {
    
            @Override
            public void run() {
                while(true) {
                    //调用重画方法
                    repaint();
                    try {
                        Thread.sleep(30);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                
            }
            
            
        }
    
    }

(7)图片加载辅助类ImageLoadUtil.java

    package com.swing.sunModel;
    
    import java.awt.Image;
    import java.awt.image.BufferedImage;
    import java.io.IOException;
    
    import javax.imageio.ImageIO;
    
    public class ImageLoadUtil {
    
        private ImageLoadUtil() {}
        
        public static Image loadImage(String path) {
            
            BufferedImage image = null;
            
            try {
                image = ImageIO.read(ImageLoadUtil.class.getResource(path));
            } catch (IOException e) {
                e.printStackTrace();
            }
    
            
            return image;
            
        }
        
    }

图片放在src下的lib文件夹下
