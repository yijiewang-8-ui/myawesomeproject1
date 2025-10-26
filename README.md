# myawesomeproject1
/**
 Visualizing Melody — Processing + Minim
 Complete visualization with styled buttons and file loading.
 
 Controls:
   - Play/Pause button OR SPACE key
   - Stop button OR 'S' key
   - Load button to choose a new file
*/

import ddf.minim.*;
import ddf.minim.analysis.*;
import java.io.File;

Minim minim;
AudioPlayer song;
FFT fft;

ArrayList<Particle> particles = new ArrayList<Particle>();
float hueOffset = 0;
boolean loaded = false;
String songName = "No song loaded";

// Button setup
Button playBtn, stopBtn, loadBtn;

void setup() {
  size(1000, 600);
  colorMode(HSB, 360, 100, 100);
  textAlign(CENTER, CENTER);

  minim = new Minim(this);

  // Try loading default song
  File f = new File(dataPath("song.mp3"));
  if (f.exists()) {
    song = minim.loadFile("song.mp3", 2048);
    fft = new FFT(song.bufferSize(), song.sampleRate());
    fft.linAverages(64);
    loaded = true;
    songName = "song.mp3";
  } else {
    fft = new FFT(1024, 44100);
    fft.linAverages(64);
  }

  // Buttons
  playBtn = new Button(40, height-60, 120, 40, "Play / Pause", color(270,80,80), color(320,80,80));
  stopBtn = new Button(180, height-60, 120, 40, "Stop", color(0,80,90), color(20,80,90));
  loadBtn = new Button(320, height-60, 120, 40, "Load File", color(200,70,80), color(250,70,80));
}

void draw() {
  drawBackgroundGradient();

  // Buttons
  playBtn.display();
  stopBtn.display();
  loadBtn.display();

  // Song name
  fill(0,0,100);
  textSize(16);
  text(songName, width/2, 20);

  if (!loaded || song == null) {
    fill(255);
    text("Load a song (put song.mp3 in data/ OR click Load File)", width/2, height/2);
    return;
  }

  if (song.isPlaying()) {
    fft.forward(song.mix);
  }

  float level = song.mix.level();
  float bass = averageFirstAverages() / 40.0;
  float treble = averageLastAverages() / 40.0;
  hueOffset = (hueOffset + 0.6) % 360;

  // Central pulse
  noStroke();
  fill(hueOffset, 80, 100, 200);
  float pulse = map(level, 0, 0.12, 80, 280);
  ellipse(width/2, height/2, pulse, pulse);

  // Spectrum bars
  int bands = fft.avgSize();
  float barW = (float) width / bands;
  for (int i = 0; i < bands; i++) {
    float h = fft.getAvg(i) * 5;
    fill((map(i, 0, bands, 0, 360) + hueOffset) % 360, 90, 95);
    rect(i * barW, height - h, barW - 2, h);
  }

  // Waveform
  stroke((hueOffset + 180) % 360, 90, 100, 200);
  noFill();
  beginShape();
  int bufSize = song.bufferSize();
  for (int i = 0; i < bufSize; i += max(1, bufSize/500)) {
    float sample = song.mix.get(i);
    float x = map(i, 0, bufSize, 0, width);
    float y = height * 0.5 + sample * 200;
    vertex(x, y);
  }
  endShape();

  // Spirals
  pushMatrix();
  translate(width/2, height/2);
  for (int s = 0; s < 3; s++) {
    noFill();
    beginShape();
    for (int i = 0; i < 90; i++) {
      float ang = i * 0.26f + frameCount * 0.02f + s * TWO_PI / 3;
      float r = i * 2.5f * (1 + bass * 0.6f);
      float x = r * cos(ang), y = r * sin(ang);
      stroke((i*4 + hueOffset + s*80) % 360, 70, 90, map(i, 0, 90, 200, 0));
      vertex(x, y);
    }
    endShape();
  }
  popMatrix();

  // Particles
  if (frameCount % 3 == 0 && level > 0.02) {
    particles.add(new Particle(width/2, height/2, hueOffset));
    if (particles.size() > 300) particles.remove(0);
  }
  for (int i = particles.size() - 1; i >= 0; i--) {
    Particle p = particles.get(i);
    p.update(level);
    p.display(bass, treble);
    if (p.isDead()) particles.remove(i);
  }
}

// Mouse clicks for buttons
void mousePressed() {
  if (playBtn.isHovering()) {
    togglePlayPause();
  } else if (stopBtn.isHovering()) {
    stopSong();
  } else if (loadBtn.isHovering()) {
    selectInput("Select an audio file:", "fileSelected");
  }
}

// Keyboard controls
void keyPressed() {
  if (key == ' ') togglePlayPause();
  if (key == 's' || key == 'S') stopSong();
}

// File chooser callback
void fileSelected(File selection) {
  if (selection == null) return;
  if (song != null) song.close();
  song = minim.loadFile(selection.getAbsolutePath(), 2048);
  fft = new FFT(song.bufferSize(), song.sampleRate());
  fft.linAverages(64);
  loaded = true;
  songName = selection.getName();
  song.pause();
  song.rewind();
}

// Play/Pause
void togglePlayPause() {
  if (song == null) return;
  if (song.isPlaying()) song.pause();
  else song.play();
}

// Stop (pause + rewind)
void stopSong() {
  if (song == null) return;
  song.pause();
  song.rewind();
}

// Release audio on exit
void stop() {
  if (song != null) song.close();
  if (minim != null) minim.stop();
  super.stop();
}

// Draw gradient background
void drawBackgroundGradient() {
  color c1 = color(265, 60, 18);
  color c2 = color(245, 50, 8);
  color c3 = color(340, 85, 28);
  for (int y = 0; y < height; y++) {
    float t = map(y, 0, height, 0, 1);
    if (t < 0.6f) {
      float tt = map(t, 0, 0.6f, 0, 1);
      stroke(lerpColor(c1, c2, tt));
    } else {
      float tt = map(t, 0.6f, 1, 0, 1);
      stroke(lerpColor(c2, c3, tt));
    }
    line(0, y, width, y);
  }
}

// ---------- Button Class ----------
class Button {
  int x, y, w, h;
  String label;
  color c1, c2;

  Button(int x, int y, int w, int h, String label, color c1, color c2) {
    this.x=x; this.y=y; this.w=w; this.h=h;
    this.label=label; this.c1=c1; this.c2=c2;
  }

  void display() {
    for (int j=0; j<h; j++) {
      float t = map(j, 0, h, 0, 1);
      stroke(lerpColor(c1, c2, t));
      line(x, y+j, x+w, y+j);
    }
    noStroke();
    fill(0,0,0,100);
    rect(x,y,w,h,8);
    fill(0,0,100);
    textSize(14);
    text(label, x+w/2, y+h/2);
  }

  boolean isHovering() {
    return mouseX >= x && mouseX <= x+w && mouseY >= y && mouseY <= y+h;
  }
}

// ---------- Particle ----------
class Particle {
  float x, y, vx, vy, alpha, size, hue;
  Particle(float sx, float sy, float baseHue) {
    x=sx; y=sy;
    vx=random(-2.5,2.5);
    vy=random(-2.5,2.5);
    alpha=255;
    size=random(3,10);
    hue=(baseHue+random(-20,40))%360;
  }
  void update(float level) {
    x+=vx*(1+level*6.0);
    y+=vy*(1+level*6.0);
    alpha-=3+level*10;
  }
  void display(float bass,float treble) {
    noStroke();
    fill((hue+bass*80)%360,90,100,alpha);
    ellipse(x,y,size*(1+treble*3),size*(1+treble*3));
  }
  boolean isDead(){ return alpha<=0; }
}

// Helpers for bass/treble
float averageFirstAverages() {
  int n = max(1, min(6, fft.avgSize()/6));
  float sum=0;
  for (int i=0;i<n;i++) sum+=fft.getAvg(i);
  return sum/n;
}
float averageLastAverages() {
  int n = max(1, min(6, fft.avgSize()/6));
  float sum=0;
  int start=fft.avgSize()-n;
  for (int i=start;i<fft.avgSize();i++) sum+=fft.getAvg(i);
  return sum/n;
}
