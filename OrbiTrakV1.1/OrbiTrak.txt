
# # # # # # # # # # # # # # # # # # # # # # # #
# Name: Austin Kidwell
# Version: 1.1
# Date: 10/29/2021
# # # # # # # # # # # # # # # # # # # # # # # #

import os
import sys
import matplotlib.pyplot as plt
import matplotlib.animation as animation
from matplotlib.animation import FuncAnimation
import matplotlib as mpl
from PyQt5 import uic, QtWidgets
from PyQt5.QtCore import QObject, QThread, pyqtSignal, pyqtSlot
from PyQt5.QtWidgets import QFileDialog
from PyQt5.QtMultimedia import QMediaContent, QMediaPlayer
from PyQt5.QtMultimediaWidgets import QVideoWidget
from PyQt5.Qt import QUrl
import math
import ctypes
import subprocess

# import pdb
# pdb.set_trace()

qtCreatorFile = "OTmenu.ui"  # Enter file here.

Ui_MainWindow, QtBaseClass = uic.loadUiType(qtCreatorFile)

Rox = []  # store gear path
Roy = []
Rcx = []  # store vacuum cup center
Rcy = []
Rcx2 = []  # store approaching cup center
Rcy2 = []
Ox = []  # store offset
Oy = []
Wx = []  # store width
Wy = []
Lx = []  # store length
Ly = []
spindles = 0


class Worker(QObject):  # worker class to be used as additional thread (fix gui freezes)
    finished = pyqtSignal()  # signals to communicate worker thread with main thread
    progress = pyqtSignal(int)

    @pyqtSlot()
    def run(self):  # function that worker thread is going to run (main thread used in gui)
        """Long-running task."""
        xmin = min(Lx)  # used to find bounds for animation
        ymin = min(Ly)
        xmax = max(Lx)
        ymax = max(Ly)
        if xmin > min(Wx):
            xmin = min(Wx)
        if ymin > min(Wy):
            ymin = min(Wy)
        if xmax < max(Wx):
            xmax = max(Wx)
        if ymax < max(Rcy):
            ymax = max(Rcy)
        if ymax < max(Wy):
            ymax = max(Wy)

        plt.ioff()  # turn plot display off
        # fig = plt.figure()  # plot
        fig, ax = plt.subplots()
        canvas_width, canvas_height = fig.canvas.get_width_height()  # Obtain length & width for cmdstring

        plt.suptitle(f"Orbi Trak ({spindles} spindles)", fontsize=14)  # name and bounds
        plt.plot(xmin, ymin)  # plot max & min (needed to set view area for animation)
        plt.plot(xmax, ymax)

        plt.plot(Rox, Roy, color='tab:gray')  # set labels and colors for graph/animation
        graph, = plt.plot(Rcx[0], Rcy[0], '--', color='tab:purple', label='Picking Cup')
        graph2, = plt.plot(Rcx2[0], Rcy2[0], '-.', color='r', label='Approaching Cup')
        graph3, = plt.plot(Wx[0], Wy[0], color='g', label='Width')
        graph4, = plt.plot(Lx[0], Ly[0], color='b', label='Length')
        plt.grid(color='#7171C6', linestyle='-', linewidth=.2)  # light grey)
        plt.autoscale(enable=True, axis='both', tight=None)
        ax.set_aspect('equal')
        plt.legend()

        def update(frame):  # animation data
            frame += 1
            frame *= 4
            graph.set_data(Rcx[:frame], Rcy[:frame])
            graph2.set_data(Rcx2[:frame], Rcy2[:frame])
            graph3.set_data(Wx[:frame], Wy[:frame])
            graph4.set_data(Lx[:frame], Ly[:frame])
            return [graph, graph2, graph3, graph4, ]

        outf = 'OT.mp4'  # commands for subprocess
        cmdstring = ('ffmpeg\\bin\\ffmpeg.exe',
                     '-y', '-r', '20',  # overwrite, 20fps
                     '-s', '%dx%d' % (canvas_width, canvas_height),  # size of image string
                     '-pix_fmt', 'argb',  # format
                     '-f', 'rawvideo', '-i', '-',  # tell ffmpeg to expect raw video from the pipe
                     '-vcodec', 'mpeg4', outf)  # output encoding
        p = subprocess.Popen(cmdstring, stdin=subprocess.PIPE)
        # Draw frames and write to the pipe
        for frame in range(120):
            # draw the frame
            update(frame)
            fig.canvas.draw()
            # extract the image as an ARGB string
            string = fig.canvas.tostring_argb()
            # write to pipe
            p.stdin.write(string)
            # update progress bar
            self.progress.emit(frame)
        # Finish up
        p.communicate()

        plt.close(fig)
        self.finished.emit()


class OrbiTrak(QtWidgets.QMainWindow, Ui_MainWindow):
    def __init__(self):
        QtWidgets.QMainWindow.__init__(self)  # Ui set-up
        Ui_MainWindow.__init__(self)
        self.setupUi(self)
        self.player = QMediaPlayer()
        self.player.setVideoOutput(self.widgetPlayer_1)  # widget for video output
        self.widgetPlayer = QVideoWidget()  # Video display widget
        self.player.durationChanged.connect(self.getDuration)  # progress bar
        self.player.positionChanged.connect(self.getPosition)
        self.progressBar_1.sliderMoved.connect(self.updatePosition)
        self.Rox = []  # store gear path
        self.Roy = []
        self.Rcx = []  # store vacuum cup center
        self.Rcy = []
        self.Rcx2 = []  # store approaching cup center
        self.Rcy2 = []
        self.Ox = []  # store offset
        self.Oy = []
        self.Wx = []  # store width
        self.Wy = []
        self.Lx = []  # store length
        self.Ly = []
        self.radius = 0  # values that alter motion path
        self.vbradius = 0
        self.offset = 0
        self.length = 0
        self.width = 0
        self.spindles = 0
        # Initialize a QThread object
        self.thread = []  # QThread()
        # Initialize a worker object
        self.worker = []  # Worker()

    def start(self):
        self.Rox = []  # store gear path, clear so multiple different runs don't combine
        self.Roy = []
        self.Rcx = []  # store vacuum cup center
        self.Rcy = []
        self.Rcx2 = []  # store approaching cup center
        self.Rcy2 = []
        self.Ox = []  # store offset
        self.Oy = []
        self.Wx = []  # store width
        self.Wy = []
        self.Lx = []  # store length
        self.Ly = []
        try:
            self.radius = float(self.lineEditRadius.text())  # get user info
            self.vbradius = self.radius / 2
            self.lineEditVBRadius.setText(str(self.vbradius))
            self.offset = float(self.lineEditOffset.text())
            self.length = float(self.lineEditLength.text())
            self.width = float(self.lineEditWidth.text())
            self.spindles = int(self.lineEditSpindle.text())
        except Exception as e:  # block wrong data
            print('No/Wrong Values Entered')
            print('Catch: ', e.__class__)
            return 1
        if self.radius < 1 or self.offset < 0 or self.offset > self.length or self.width < 1 or \
                int(self.length) < 1 or self.spindles < 2:  # handle negative values
            return 1
        self.Cycle()

    def Cycle(self):  # loop through motion and get values
        ang = 0.5
        Bc = (70 * math.pi / 180)
        Bh = (70 * math.pi / 180)
        N = Bh / Bc
        H = 90
        Hh = (H * 4 * N) / (N * 4 + math.pi)
        Hc = Hh * math.pi / 4 * N
        for i in range(720):  # loop 1 cycle in cartoon feeder
            # print('Angle: ', ang)
            an = ang
            ang = math.radians(ang)
            if an > 290:  # get spline and theta
                ang2 = an - 290
                ang2 = math.radians(ang2)
                sply = Hc * (1 - (ang2 / Bc) - (1 / math.pi) * (math.sin((math.pi * ang2) / Bc)))
                hyp = (2 * an) - sply
            elif an > 220:
                ang2 = an - 220
                ang2 = math.radians(ang2)
                sply = Hh * (math.cos((math.pi * ang2) / (2 * Bh))) + Hc
                hyp = (2 * an) - sply
            elif an > 150:
                ang2 = an - 150
                ang2 = math.radians(ang2)
                sply = Hh * (math.sin((math.pi * ang2) / (2 * Bh))) + Hc
                hyp = (2 * an) - sply
            elif an > 80:
                ang2 = an - 80
                ang2 = math.radians(ang2)
                sply = Hc * ((ang2 / Bc) - (1 / math.pi) * (math.sin(math.pi * (ang2 / Bc))))
                hyp = (2 * an) - sply
            else:
                hyp = (2 * an)

            # print('Theta: ', hyp)
            theta = math.radians(hyp)
            rx = self.radius * ((1 - math.tan(-ang / 2) ** 2) / (1 + math.tan(-ang / 2) ** 2))
            ry = self.radius * ((2 * math.tan(-ang / 2)) / (1 + math.tan(-ang / 2) ** 2))
            self.Rox.append(rx)  # set gear path data
            self.Roy.append(ry)
            vcx = self.vbradius * ((1 - math.tan(theta / 2) ** 2) / (1 + math.tan(theta / 2) ** 2))
            vcy = self.vbradius * ((2 * math.tan(theta / 2)) / (1 + math.tan(theta / 2) ** 2))
            self.Rcx.append(vcx + rx)  # set vacuum cup center data
            self.Rcy.append(vcy + ry)

            hyp2 = hyp + 90  # angle for offset and width
            hyp3 = hyp - 90  # angle for length
            theta2 = math.radians(hyp2)
            theta3 = math.radians(hyp3)

            ofx = self.offset * ((1 - math.tan(theta2 / 2) ** 2) / (1 + math.tan(theta2 / 2) ** 2))
            ofy = self.offset * ((2 * math.tan(theta2 / 2)) / (1 + math.tan(theta2 / 2) ** 2))
            self.Ox.append(ofx + self.Rcx[i])  # get offset points
            self.Oy.append(ofy + self.Rcy[i])
            wdx = (self.offset + self.width) * ((1 - math.tan(theta2 / 2) ** 2) / (1 + math.tan(theta2 / 2) ** 2))
            wdy = (self.offset + self.width) * ((2 * math.tan(theta2 / 2)) / (1 + math.tan(theta2 / 2) ** 2))
            self.Wx.append(wdx + self.Rcx[i])  # get width points
            self.Wy.append(wdy + self.Rcy[i])
            leng = self.length - self.offset
            lnx = leng * ((1 - math.tan(theta3 / 2) ** 2) / (1 + math.tan(theta3 / 2) ** 2))
            lny = leng * ((2 * math.tan(theta3 / 2)) / (1 + math.tan(theta3 / 2) ** 2))
            self.Lx.append(lnx + self.Rcx[i])  # get length points
            self.Ly.append(lny + self.Rcy[i])

            # print()
            ang = an
            ang += 0.5
        self.Rcx, self.Rcy = self.rotate(30, self.Rcx, self.Rcy, 720)
        self.Wx, self.Wy = self.rotate(30, self.Wx, self.Wy, 240)
        self.Lx, self.Ly = self.rotate(30, self.Lx, self.Ly, 240)
        self.Display()

    def rotate(self, deg, xlist, ylist, frames):  # rotate motion path
        angle = math.radians(deg)
        x = []
        y = []
        for i in range(frames):
            x.append(xlist[i] * math.cos(angle) - ylist[i] * math.sin(angle))
            y.append(xlist[i] * math.sin(angle) + ylist[i] * math.cos(angle))
        return x, y

    def Display(self):
        distWid = []
        distLen = []
        collW = False
        collL = False
        self.GetGear2()

        for i in range(len(self.Wx)):  # calc distance
            distWid.append((((self.Rcx2[i] - self.Wx[i]) ** 2) + ((self.Rcy2[i] - self.Wy[i]) ** 2) ** 0.5))
            distLen.append((((self.Rcx2[i] - self.Lx[i]) ** 2) + ((self.Rcy2[i] - self.Ly[i]) ** 2) ** 0.5))
            if ((self.Rcx2[i] + 5) > self.Wx[i]) & ((self.Rcy2[i] - 25) < self.Wy[i]):  # collision test width
                collW = True
            elif ((self.Rcx2[i] + 5) > self.Lx[i]) & ((self.Rcy2[i] - 25) < self.Ly[i]):  # collision test length
                collL = True

        ctypes.windll.user32.MessageBoxW(0, f'Min distance from width and cup: {min(distWid)} (mm)\n' +
                                         f'Min distance from length and cup: {min(distLen)} (mm)\n' +
                                         f'Collision between width and cup: {collW}\n' +
                                         f'Collision between length and cup: {collL}\n', 'Results', 0)

    def GetGear2(self):  # crate cup line behind current one
        x = []
        y = []
        total = 719
        delay = (total + 1) / self.spindles
        start = total - delay + 1
        for i in range(len(self.Rcx)):
            temp = int(i + start)
            if temp > total:
                temp -= total
            x.append(self.Rcx[temp])
            y.append(self.Rcy[temp])
        self.Rcx2 = x
        self.Rcy2 = y

    def save(self):
        if len(self.Rox) == 0:  # prevents saving and loading nothing
            return 1

        global Rox, Roy, Rcx, Rcy, Rcx2, Rcy2, Ox, Oy, Wx, Wy, Lx, Ly, spindles  # copy variables for threads
        Rox = self.Rox  # store gear path
        Roy = self.Roy
        Rcx = self.Rcx  # store vacuum cup center
        Rcy = self.Rcy
        Rcx2 = self.Rcx2  # store approaching cup center
        Rcy2 = self.Rcy2
        Ox = self.Ox  # store offset
        Oy = self.Oy
        Wx = self.Wx  # store width
        Wy = self.Wy
        Lx = self.Lx  # store length
        Ly = self.Ly
        spindles = self.spindles

        # Create a QThread object
        self.thread = QThread()
        # Create a worker object
        self.worker = Worker()
        # Move worker to the thread
        self.worker.moveToThread(self.thread)
        # Connect signals and slots
        self.thread.started.connect(self.worker.run)
        self.worker.finished.connect(self.thread.quit)
        self.worker.finished.connect(self.worker.deleteLater)
        self.thread.finished.connect(self.thread.deleteLater)
        self.worker.progress.connect(self.reportProgress)
        # Start the thread
        self.thread.start()

        # Disable all buttons except cancel while worker thread is functioning
        self.startBtn_1.setEnabled(False)
        self.thread.finished.connect(
            lambda: self.startBtn_1.setEnabled(True)
        )
        self.saveButton_1.setEnabled(False)
        self.thread.finished.connect(
            lambda: self.saveButton_1.setEnabled(True)
        )
        self.playBtn_1.setEnabled(False)
        self.thread.finished.connect(
            lambda: self.playBtn_1.setEnabled(True)
        )
        # Call load after worker thread is finished
        self.thread.finished.connect(self.load)

    def reportProgress(self, value):  # Used to update loading bar
        self.progressBar.setValue(value)

    def cancel(self):
        self.close()

    def load(self):
        w = window.geometry().width()               # get window size
        h = window.geometry().height()

        self.setupUi(self)  # set-up to update
        self.player = QMediaPlayer()
        self.player.setVideoOutput(self.widgetPlayer_1)

        self.player.durationChanged.connect(self.getDuration)  # sliding bar
        self.player.positionChanged.connect(self.getPosition)
        self.progressBar_1.sliderMoved.connect(self.updatePosition)

        self.player.setMedia(QMediaContent(QUrl.fromLocalFile(r'OT.mp4')))  # load and play
        self.player.play()
        app.exec_()  # need to update

        window.resize(w, h)         # keep window same size
        self.lineEditRadius.setText(str(self.radius))  # keep values in ui
        self.lineEditVBRadius.setText(str(self.vbradius))
        self.lineEditOffset.setText(str(self.offset))
        self.lineEditLength.setText(str(self.length))
        self.lineEditWidth.setText(str(self.width))
        self.lineEditSpindle.setText(str(self.spindles))

    def playvid(self):  # pause and play video
        if self.player.state() == 1:
            self.player.pause()
        else:
            self.player.play()

    def getDuration(self, d):  # d is total time in ms
        self.progressBar_1.setRange(0, d)
        self.progressBar_1.setEnabled(True)
        self.displayTime(d)

    def getPosition(self, p):
        self.progressBar_1.setValue(p)
        self.displayTime(self.progressBar_1.maximum() - p)

    def displayTime(self, ms):  # show time and convert from ms
        minutes = int(ms / 60000)
        seconds = int((ms - minutes * 60000) / 1000)
        self.labelTime_1.setText('{}:{}'.format(minutes, seconds))

    def updatePosition(self, v):
        self.player.setPosition(v)
        self.displayTime(self.progressBar_1.maximum() - v)


if __name__ == "__main__":
    app = QtWidgets.QApplication(sys.argv)
    window = OrbiTrak()
    window.show()
    sys.exit(app.exec_())
