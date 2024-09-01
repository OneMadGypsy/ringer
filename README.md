```python3
import pygame, sys, time, json
import numpy as np
from functools import partial


class RingGame:
    BGCOLOR    = 0x000000
    HIGHLIGHT  = 0xFFFFFF
    SAMPLERATE = 44100
    VOLUME     = .5
    DURATION   = .5
    
    @staticmethod
    def generate_note(frequency:float, duration:float, volume:float=1.0):
        t     = np.linspace(0, duration, int(RingGame.SAMPLERATE * duration), False)
        tone  = np.sin(frequency * np.pi * 2 * t) 
        tone *= 32767 / np.max(np.abs(tone))
        tone  = tone.astype(np.int16)
        sound = pygame.mixer.Sound(tone)
        sound.set_volume(volume)
        sound.play(fade_ms=10)
        
    @staticmethod
    def make_keymap(*args) -> dict:
       freqs = {
           "C" : (16.35160,0xFf0000),
           "C#": (17.32391,0x00FF00),
           "D" : (18.35405,0x0000FF),
           "D#": (19.44544,0xFFFF00),
           "E" : (20.60172,0x00FFFF),
           "F" : (21.82676,0x7F0000),
           "F#": (23.12465,0x007F00),
           "G" : (24.49971,0x00007F),
           "G#": (25.95654,0xFF7F00),
           "A" : (27.50000,0x00FF7F),
           "A#": (29.13524,0x7FFF7F),
           "B" : (30.86771,0xFF7FFF),
       }
       
       data = {}
       
       for (key, note, octave) in args:
          if (v := freqs.get(note, None)) is None:
              raise ValueError('key does not exist')
              
          data[key] = (v[0] * 2 ** octave) or v[0], v[1]
               
       return data   
        
    def __init__(self):
        pygame.init()
        pygame.mixer.init()
        pygame.display.set_caption('Musical Rings')
        
        self.clock  = pygame.time.Clock()
        self.screen = pygame.display.set_mode((1200, 800))
        
        
        self.keymap = RingGame.make_keymap((pygame.K_a, 'G' , 2),
                                           (pygame.K_s, 'A' , 2),
                                           (pygame.K_d, 'B' , 2),
                                           (pygame.K_f, 'C' , 3),
                                           (pygame.K_g, 'D' , 3),
                                           (pygame.K_h, 'E' , 3),
                                           (pygame.K_j, 'F#', 3),
                                           (pygame.K_k, 'G' , 3))
        
        self.circle = partial(pygame.draw.circle, self.screen)
        
        self.pressed = []
        self.playing = True
        
    def run(self):
        while self.playing:
            self.check_events()
            self.update_screen()
            self.clock.tick(60)
        
    def check_events(self):
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                self.playing = False
                pygame.mixer.quit()
                pygame.quit()
                sys.exit()    
            elif event.type == pygame.KEYDOWN:
                if tone := self.keymap.get(event.key, (0,0,0))[0]:
                    RingGame.generate_note(tone, RingGame.DURATION, RingGame.VOLUME)
                    if event.key not in self.pressed:
                        self.pressed.append(event.key)   
            elif event.type == pygame.KEYUP:
                self.pressed.remove(event.key)
    
    def update_screen(self):
        self.screen.fill(RingGame.BGCOLOR)
        self.make_rings()
        pygame.display.flip()

    def make_rings(self):
        w,h = self.screen.get_width()/(len(self.keymap)+1), self.screen.get_height()/2
        for i,(k,(_,c)) in enumerate(self.keymap.items(), 1):
            if k in self.pressed:
                self.circle(color=RingGame.HIGHLIGHT, center=(w*i, h), radius=w/2.1)
            self.circle(color=c, center=(w*i, h), radius=w/2.2)
       
        
if __name__ == "__main__":
    RingGame().run()
```
