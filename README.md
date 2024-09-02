```python3
import pygame, sys, time
import numpy as np
from functools   import partial
from enum        import IntEnum
from collections import deque
from scipy.fft import fft, ifft

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
            

class Ringer:
    BGCOLOR    = 0x000000
    HIGHLIGHT  = 0xFFFFFF
    SAMPLERATE = 44100
    VOLUME     = .5
    DURATION   = .5
    
    # synth: default "piano"
    HARMONICS = 1.0, 1.5, 0.3, 0.2, 0.1
    ATTACK    = 0.05
    DECAY     = 0.1
    RELEASE   = 0.1
    SUSTAIN   = 0.7
    
    # effects
    REVERB      = 0.0
    ECHODELAY   = 0.0
    ECHODECAY   = 0.0
    CHORUSDELAY = 0.0
    CHORUSDEPTH = 0.0
    CHORUSRATE  = 0.0
    CHORUSMIX   = 0.0
    
    @staticmethod
    def __apply_reverb(signal, reverb_time:float):
        # generate a simple impulse response for reverb
        impulse_response_length = int(Ringer.SAMPLERATE * reverb_time)
        impulse_response = np.linspace(1, 0, impulse_response_length)
        
        signal_length = len(signal)
        impulse_response_length = len(impulse_response)
        fft_length = 2**int(np.ceil(np.log2(signal_length + impulse_response_length - 1)))
        
        # perform convolution
        signal_fft = fft(signal, n=fft_length)
        impulse_response_fft = fft(impulse_response, n=fft_length)
        
        reverb_fft = signal_fft * impulse_response_fft
        reverb_signal = np.real(ifft(reverb_fft))
        
        return reverb_signal[:signal_length]
        
    @staticmethod
    def __apply_echo(signal, delay_time, decay_factor):
        # calculate the number of samples
        delay_samples = int(Ringer.SAMPLERATE * delay_time)
        
        # generate an array of sample points
        output_signal = np.zeros(len(signal) + delay_samples)
        
        # copy the original signal to the start of the output array
        output_signal[:len(signal)] = signal
        
        # create the echo signal
        echo_signal = np.zeros_like(output_signal)
        echo_signal[delay_samples:] = signal * decay_factor
        
        # add the echo signal to the output signal
        output_signal += echo_signal
        
        return output_signal
        
    @staticmethod
    def __apply_chorus(signal, delay_time:float, depth:float, rate:float, mix:float):
        # calculate the number of samples
        max_delay_samples = int(Ringer.SAMPLERATE * delay_time)
        
        # create an output array
        output_signal = np.zeros(len(signal) + max_delay_samples)
        
        # generate modulation LFO for delay
        t = np.arange(len(signal)) / Ringer.SAMPLERATE
        lfo = depth * max_delay_samples * (np.sin(2 * np.pi * rate * t) / 2 + 0.5)
        
        # apply the chorus effect
        for i in range(len(signal)):
            delay_samples = int(lfo[i])
            if i + delay_samples < len(signal):
                output_signal[i] = signal[i] * (1 - mix) + signal[i + delay_samples] * mix
        
        output_signal = output_signal[:len(signal)]
        
        return output_signal
        
    @staticmethod     
    def synth(frequency:float, duration:float, volume:float=1.0, **kwargs):
        # calculate the number of samples
        num_samples = int(Ringer.SAMPLERATE * duration)
        
        # generate an array of sample points
        t = np.linspace(0, duration, num_samples, endpoint=False)
        
        # generate a complex waveform with harmonics
        harmonics = kwargs.get("harmonics" , Ringer.HARMONICS)
        wave_points = np.sum([
            h * np.sin(2 * np.pi * frequency * (i + 1) * t)
            for i, h in enumerate(harmonics)
        ], axis=0)
        
        # normalize
        
        # ADSR envelope parameters
        attack_time   = kwargs.get("attack"  , Ringer.ATTACK)
        decay_time    = kwargs.get("decay"   , Ringer.DECAY)
        release_time  = kwargs.get("release" , Ringer.RELEASE)
        sustain_level = kwargs.get("sustain" , Ringer.SUSTAIN)
        
        # effect parameters
        reverb_time   = kwargs.get("reverb"       , Ringer.REVERB)
        echo_delay    = kwargs.get("echo_delay"   , Ringer.ECHODELAY)
        echo_decay    = kwargs.get("echo_decay"   , Ringer.ECHODECAY)
        chorus_delay  = kwargs.get("chorus_delay" , Ringer.CHORUSDELAY)
        chorus_depth  = kwargs.get("chorus_depth" , Ringer.CHORUSDEPTH)
        chorus_rate   = kwargs.get("chorus_rate"  , Ringer.CHORUSRATE)
        chorus_mix    = kwargs.get("chorus_mix"   , Ringer.CHORUSMIX)
        
        # create envelope arrays
        attack_samples  = int(attack_time  * Ringer.SAMPLERATE)
        decay_samples   = int(decay_time   * Ringer.SAMPLERATE)
        release_samples = int(release_time * Ringer.SAMPLERATE) 
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
        wave_points *= volume
           
        # add effects
        if reverb_time > 0:
            wave_points = Ringer.__apply_reverb(wave_points, reverb_time)
            
        if echo_delay * echo_decay:
            wave_points = Ringer.__apply_echo(wave_points, echo_delay, echo_decay)
            
        if chorus_delay * chorus_depth * chorus_rate * chorus_mix:
            wave_points = Ringer.__apply_chorus(wave_points, chorus_delay, chorus_depth, chorus_rate, chorus_mix)
            
        # normalize and format to 16-bit PCM format 
        wave_points /= np.max(np.abs(wave_points))
        wave_points = np.int16(wave_points * 32767)
        
        return wave_points
        
    @staticmethod
    def make_keymap(*args, **kwargs) -> dict:
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
              
          #precompute all waves - otherwise it's too slow
          freq = (v[0] * 2 ** octave) or v[0]
          data[key] = Ringer.synth(freq, Ringer.DURATION, Ringer.VOLUME, **kwargs), v[1]
               
       return data   
        
    def __init__(self):
        pygame.init()
        pygame.mixer.init()
        pygame.display.set_caption('Ringer')
        
        self.clock  = pygame.time.Clock()
        self.screen = pygame.display.set_mode((1200, 800))
        
        # G Major Ionian scale
        scale  = Scales.scale('G', Scales.MAJOR, Modes.Ionian, 2)
        
        # keys to correspond to each scale note
        keys   = (pygame.K_a, pygame.K_s, pygame.K_d, pygame.K_f,
                  pygame.K_g, pygame.K_h, pygame.K_j, pygame.K_k,)
        
        # synth effects properties
        self.effects = dict(
            reverb       = 0.5, 
            echo_delay   = 0.3, 
            echo_decay   = 0.5,
            chorus_delay = 0.01,
            chorus_depth = 0.5,
            chorus_rate  = 1,
            chorus_mix   = 0.5,
        )
        
        # the length of this keymap will determine how many circles are drawn
        self.keymap = Ringer.make_keymap(*zip(keys, scale), **self.effects)
        
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
                if (data := self.keymap.get(event.key, None)) is not None:
                    pygame.mixer.Sound(data[0]).play()
                    
                    # keep track of pressed keys
                    if event.key not in self.pressed:
                        self.pressed.append(event.key) 
                        
            elif event.type == pygame.KEYUP:
                if event.key in self.pressed:
                    # key is no longer pressed, remove it 
                    self.pressed.remove(event.key)
    
    def update_screen(self):
        self.screen.fill(Ringer.BGCOLOR)
        self.make_rings()
        pygame.display.flip()

    def make_rings(self):
        # ring count and sizes are dynamically determined by the length of `keymap`
        w,h = self.screen.get_width()/(len(self.keymap)+1), self.screen.get_height()/2
        for i,(k,(_,c)) in enumerate(self.keymap.items(), 1):
            
            # draw a highlight ring around circles that correspond to pressed keys
            if k in self.pressed:
                self.circle(color=Ringer.HIGHLIGHT, center=(w*i, h), radius=w/2.1)
            
            self.circle(color=c, center=(w*i, h), radius=w/2.2)
       
        
if __name__ == "__main__":
    Ringer().run()

```
