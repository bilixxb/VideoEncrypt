import sys
import cv2
import numpy as np
from PyQt5.QtWidgets import (QApplication, QMainWindow, QLabel, QPushButton, 
                             QFileDialog, QVBoxLayout, QWidget, QProgressBar,
                             QHBoxLayout, QMessageBox)
from PyQt5.QtCore import Qt, QThread, pyqtSignal
from PyQt5.QtGui import QIcon, QPixmap, QImage

class VideoProcessor(QThread):
    progress_updated = pyqtSignal(int)
    finished = pyqtSignal(str, bool)  # (message, is_error)
    
    def __init__(self, input_path, output_path, seed, mode, parent=None):
        super().__init__(parent)
        self.input_path = input_path
        self.output_path = output_path
        self.seed = seed
        self.mode = mode  # 'encrypt' or 'decrypt'
        self.canceled = False
        
    def run(self):
        try:
            cap = cv2.VideoCapture(self.input_path)
            if not cap.isOpened():
                self.finished.emit("无法打开视频文件", True)
                return
                
            # 获取视频属性
            fps = cap.get(cv2.CAP_PROP_FPS)
            width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
            height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
            total_frames = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
            
            # 设置视频编码器（使用H264编码）
            fourcc = cv2.VideoWriter_fourcc(*'H264')
            out = cv2.VideoWriter(self.output_path, fourcc, fps, (width, height))
            
            # 初始化随机数生成器
            np.random.seed(self.seed)
            
            frame_count = 0
            while cap.isOpened():
                if self.canceled:
                    break
                    
                ret, frame = cap.read()
                if not ret:
                    break
                
                # 处理当前帧
                if self.mode == 'encrypt':
                    processed_frame = self.encrypt_frame(frame)
                else:
                    processed_frame = self.decrypt_frame(frame)
                
                out.write(processed_frame)
                frame_count += 1
                progress = int((frame_count / total_frames) * 100)
                self.progress_updated.emit(progress)
            
            cap.release()
            out.release()
            
            if self.canceled:
                self.finished.emit("操作已取消", True)
            else:
                self.finished.emit(f"操作成功完成！保存于: {self.output_path}", False)
                
        except Exception as e:
            self.finished.emit(f"发生错误: {str(e)}", True)
    
    def encrypt_frame(self, frame):
        # 生成相同尺寸的随机帧
        random_frame = np.random.randint(0, 256, frame.shape, dtype=np.uint8)
        # 异或加密
        return cv2.bitwise_xor(frame, random_frame)
    
    def decrypt_frame(self, frame):
        # 解密使用相同的加密方法（异或两次等于原数据）
        return self.encrypt_frame(frame)
    
    def cancel(self):
        self.canceled = True

class VideoEncryptorApp(QMainWindow):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("视频加密解密工具")
        self.setGeometry(100, 100, 600, 400)
        #self.setWindowIcon(QIcon(self.style().standardPixmap(QStyle.SP_FileDialogInfoView)))
        
        # 初始化UI
        self.init_ui()
        
        # 初始化变量
        self.input_file = ""
        self.output_file = ""
        self.processor = None
        self.seed = 12345  # 默认加密种子
        
    def init_ui(self):
        # 创建主部件和布局
        main_widget = QWidget()
        main_layout = QVBoxLayout()
        
        # 标题标签
        title_label = QLabel("视频加密解密工具")
        title_label.setStyleSheet("font-size: 18px; font-weight: bold;")
        title_label.setAlignment(Qt.AlignCenter)
        
        # 说明标签
        desc_label = QLabel("将视频加密为随机像素视频，或解密还原原始视频")
        desc_label.setAlignment(Qt.AlignCenter)
        
        # 输入文件选择
        input_layout = QHBoxLayout()
        self.input_label = QLabel("未选择文件")
        self.input_label.setStyleSheet("border: 1px solid gray; padding: 5px;")
        input_layout.addWidget(self.input_label)
        
        browse_btn = QPushButton("选择视频")
        browse_btn.clicked.connect(self.browse_input)
        input_layout.addWidget(browse_btn)
        
        # 操作按钮
        btn_layout = QHBoxLayout()
        self.encrypt_btn = QPushButton("加密视频")
        self.encrypt_btn.setEnabled(False)
        self.encrypt_btn.clicked.connect(lambda: self.process_video('encrypt'))
        btn_layout.addWidget(self.encrypt_btn)
        
        self.decrypt_btn = QPushButton("解密视频")
        self.decrypt_btn.setEnabled(False)
        self.decrypt_btn.clicked.connect(lambda: self.process_video('decrypt'))
        btn_layout.addWidget(self.decrypt_btn)
        
        # 进度条
        self.progress_bar = QProgressBar()
        self.progress_bar.setVisible(False)
        
        # 取消按钮
        self.cancel_btn = QPushButton("取消")
        self.cancel_btn.setVisible(False)
        self.cancel_btn.clicked.connect(self.cancel_processing)
        
        # 添加组件到主布局
        main_layout.addWidget(title_label)
        main_layout.addWidget(desc_label)
        main_layout.addSpacing(20)
        main_layout.addLayout(input_layout)
        main_layout.addLayout(btn_layout)
        main_layout.addSpacing(20)
        main_layout.addWidget(self.progress_bar)
        main_layout.addWidget(self.cancel_btn)
        main_layout.addStretch()
        
        main_widget.setLayout(main_layout)
        self.setCentralWidget(main_widget)
    
    def browse_input(self):
        file_path, _ = QFileDialog.getOpenFileName(
            self, "选择视频文件", "", 
            "视频文件 (*.mp4 *.avi *.mov *.mkv);;所有文件 (*.*)"
        )
        
        if file_path:
            self.input_file = file_path
            self.input_label.setText(file_path.split('/')[-1])
            self.input_label.setToolTip(file_path)
            self.encrypt_btn.setEnabled(True)
            self.decrypt_btn.setEnabled(True)
    
    def process_video(self, mode):
        if not self.input_file:
            QMessageBox.warning(self, "警告", "请先选择视频文件")
            return
            
        # 设置输出文件路径
        default_name = "encrypted" if mode == 'encrypt' else "decrypted"
        output_path, _ = QFileDialog.getSaveFileName(
            self, "保存结果视频", f"{default_name}.mp4", 
            "MP4文件 (*.mp4);;所有文件 (*.*)"
        )
        
        if not output_path:
            return
            
        # 显示进度条
        self.progress_bar.setValue(0)
        self.progress_bar.setVisible(True)
        self.cancel_btn.setVisible(True)
        self.encrypt_btn.setEnabled(False)
        self.decrypt_btn.setEnabled(False)
        
        # 创建处理线程
        self.processor = VideoProcessor(
            self.input_file, output_path, self.seed, mode
        )
        self.processor.progress_updated.connect(self.update_progress)
        self.processor.finished.connect(self.processing_finished)
        self.processor.start()
    
    def update_progress(self, value):
        self.progress_bar.setValue(value)
    
    def processing_finished(self, message, is_error):
        self.progress_bar.setVisible(False)
        self.cancel_btn.setVisible(False)
        self.encrypt_btn.setEnabled(True)
        self.decrypt_btn.setEnabled(True)
        
        if is_error:
            QMessageBox.critical(self, "错误", message)
        else:
            QMessageBox.information(self, "完成", message)
    
    def cancel_processing(self):
        if self.processor and self.processor.isRunning():
            self.processor.cancel()
            self.processor.wait()
            self.processing_finished("操作已取消", True)

if __name__ == "__main__":
    app = QApplication(sys.argv)
    window = VideoEncryptorApp()
    window.show()
    sys.exit(app.exec_())
