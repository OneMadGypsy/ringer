```python3
import pygame, sys, time
import numpy as np
from functools   import partial
from enum        import IntEnum
from collections import deque


class Modes(IntEnum):
    Ionian     = 0
    Dorian     = 1
    Phrygian   = 2
    Lydian     = 3
    Mixolydian = 4
    Aolian     = 5
    Locrian    = 6

    
class Scales:
    NOTES = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B']
    
    # scale formulas
    MAJOR           = 2, 2, 1, 2, 2, 2, 1
    GYPSY           = 1, 3, 1, 2, 1, 3, 1
    ENIGMATIC       = 1, 3, 2, 2, 2, 1, 1
    PERSIAN         = 1, 3, 1, 1, 2, 1, 3
    NATURALMINOR    = 2, 1, 2, 2, 1, 2, 2
    MELODICMINOR    = 2, 1, 2, 2, 2, 2, 1
    HUNGARIANMINOR  = 2, 1, 3, 1, 1, 3, 1
    NEOPOLITANMINOR = 1, 2, 2, 2, 1, 2, 2

    @staticmethod
    def scale(key:str, steps:tuple|list, mode:int, octave:int) -> tuple:
        # find start note
        start = Scales.NOTES.index(key)
        
        # rotate to proper start note
        notes = deque(Scales.NOTES)
        notes.rotate(-(start + sum(steps[:mode])))
        
        # rotate steps and prime for start note
        nsteps = deque(steps)
        nsteps.rotate(-mode)
        steps = (0, *nsteps)
        
        # gather scale and proper octave
        i, oct, c, scale = 0, octave, notes.index('C'), []
        for s in steps:
            i += s
            note = notes[i%12]
            oct += all((oct==octave, i>=c, i>0))
            scale.append((note, oct))
            
        return tuple(scale)
            

class RingGame:
    BGCOLOR    = 0x000000
    HIGHLIGHT  = 0xFFFFFF
    SAMPLERATE = 44100
    VOLUME     = .5
    DURATION   = 1
    
    def generate_piano_note(frequency:float, duration:float, volume:float=1.0) -> None:
        # calculate the number of samples
        num_samples = int(RingGame.SAMPLERATE * duration)
        
        # generate an array of sample points
        t = np.linspace(0, duration, num_samples, endpoint=False)
        
        # generate a complex waveform with harmonics
        harmonics = (1.0, 0.5, 0.3, 0.2, 0.1)
        wave_points = np.sum([
            h * np.sin(2 * np.pi * frequency * (i + 1) * t)
            for i, h in enumerate(harmonics)
        ], axis=0)
        
        # normalize
        wave_points /= np.max(np.abs(wave_points))
        
        # ADSR envelope parameters
        attack_time   = 0.05
        decay_time    = 0.1
        sustain_level = 0.7
        release_time  = 0.1
        
        # create envelope arrays
        attack_samples  = int(attack_time  * RingGame.SAMPLERATE)
        decay_samples   = int(decay_time   * RingGame.SAMPLERATE)
        release_samples = int(release_time * RingGame.SAMPLERATE) 
        sustain_samples = int(num_samples - attack_samples - decay_samples - release_samples)
        
        attack  = np.linspace(0, 1, attack_samples)
        decay   = np.linspace(1, sustain_level, decay_samples)
        sustain = np.ones(sustain_samples) * sustain_level
        release = np.linspace(sustain_level, 0, release_samples)
        
        # concatenate the envelope
        envelope = np.concatenate((attack, decay, sustain, release))
        
        # ensure envelope length matches wave points length
        if len(envelope) < len(wave_points):
            envelope = np.pad(envelope, (0, len(wave_points) - len(envelope)), 'constant', constant_values=0)
        elif len(envelope) > len(wave_points):
            envelope = envelope[:len(wave_points)]
        
        # apply the envelope to the wave points
        wave_points *= envelope
           
        # 16-bit PCM format 
        wave_points = np.int16(wave_points * 32767)
        
        # create, customize and play sound
        sound = pygame.mixer.Sound(wave_points)
        sound.set_volume(volume)
        sound.play()
        
    @staticmethod
    def make_keymap(*args) -> dict:
       # we can derive all octaves from this base
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
       for (key, (note, octave)) in args:
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
        
        # G Major Phrygian scale
        scale  = Scales.scale('G', Scales.MAJOR, Modes.Phrygian, 2)
        
        # keys to correspond to each scale note
        keys   = (pygame.K_a, pygame.K_s, pygame.K_d, pygame.K_f,
                  pygame.K_g, pygame.K_h, pygame.K_j, pygame.K_k,)
        
        # the length of this keymap will determine how many circles are drawn
        self.keymap = RingGame.make_keymap(*zip(keys, scale))
        
        # makes the call to `draw.circle` a little shorter
        self.circle = partial(pygame.draw.circle, self.screen)
        
        self.pressed = []   # for tracking keypresses
        self.playing = True # game loop condition
        
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
                # find tone and generate it
                if tone := self.keymap.get(event.key, (0,0,0))[0]:
                    RingGame.generate_piano_note(tone, RingGame.DURATION, RingGame.VOLUME)
                    
                    # keep track of pressed keys
                    if event.key not in self.pressed:
                        self.pressed.append(event.key) 
                        
            elif event.type == pygame.KEYUP:
                if event.key in self.pressed:
                    # key is no longer pressed, remove it 
                    self.pressed.remove(event.key)
    
    def update_screen(self):
        self.screen.fill(RingGame.BGCOLOR)
        self.make_rings()
        pygame.display.flip()

    def make_rings(self):
        # ring count and sizes are dynamically determined by the length of `keymap`
        w,h = self.screen.get_width()/(len(self.keymap)+1), self.screen.get_height()/2
        for i,(k,(_,c)) in enumerate(self.keymap.items(), 1):
            
            # draw a highlight ring around circles that correspond to pressed keys
            if k in self.pressed:
                self.circle(color=RingGame.HIGHLIGHT, center=(w*i, h), radius=w/2.1)
            
            self.circle(color=c, center=(w*i, h), radius=w/2.2)
       
        
if __name__ == "__main__":
    RingGame().run()
```
