```python3
import pygame, sys, time, json
import numpy as np
from functools import partial


class RingGame:
    COLORS     = 0xFF0000, 0x0000FF, 0xFFFF00, 0x00FF00, 0xFF7F00, 0x7F00FF, 0x00FF7F, 0xFF007F
    BGCOLOR    = 0x000000
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
        sound.play()
        
    def __init__(self):
        pygame.init()
        pygame.mixer.init()
        pygame.display.set_caption('Musical Rings')
        
        self.clock  = pygame.time.Clock()
        self.screen = pygame.display.set_mode((1200, 800))
        
        self.ring_center = (w,_) = self.screen.get_width()/(len(RingGame.COLORS)+1), self.screen.get_height()/2
        self.circle      = partial(pygame.draw.circle, self.screen, radius=w/2.2)
        
        #load tones
        self.tones = json.load((file := open('tones.json', 'r')))
        file.close()
        
        self.keymap = { # G scale
            pygame.K_a : self.tones['G2'] ,
            pygame.K_s : self.tones['A2'] ,
            pygame.K_d : self.tones['B2'] ,
            pygame.K_f : self.tones['C3'] ,
            pygame.K_g : self.tones['D3'] ,
            pygame.K_h : self.tones['E3'] ,
            pygame.K_j : self.tones['F#3'],
            pygame.K_k : self.tones['G3'] ,
        }

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
                if tone := self.keymap.get(event.key, None):
                    RingGame.generate_note(tone, RingGame.DURATION, RingGame.VOLUME)
    
    def update_screen(self):
        self.screen.fill(RingGame.BGCOLOR)
        self.make_rings()
        pygame.display.flip()

    def make_rings(self):
        w,h = self.ring_center
        for i,c in enumerate(RingGame.COLORS, 1):
            self.circle(color=c, center=(w*i, h))
       
        
if __name__ == "__main__":
    RingGame().run()
```
