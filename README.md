Simple Description-
This project is a Smart Automatic Dustbin which automatically opens the lid when someone comes near using an Ultrasonic Sensor. It also separates waste into Wet and Dry using a Moisture Sensor. The system uses Arduino Uno and Servo Motor to rotate the waste divider. This helps in clean and hygienic waste disposal and improves waste management and recycling.
Short Working-
Object detected → Lid opens automatically
Moisture sensor checks waste type
Wet waste goes to Wet bin
Dry waste goes to Dry bin
Lid closes automatically
flowchart-Smart Automatic Dustbin
                         ┌──────────────────────┐
                         │      Start System     │
                         └────────────┬──────────┘
                                      │
                                      ▼
                         ┌──────────────────────┐
                         │  Detect Object Using  │
                         │  Ultrasonic Sensor    │
                         └────────────┬──────────┘
                                      │
                     ┌────────────────▼────────────────┐
                     │  Is Distance ≤ 15 cm Detected ?  │
                     └────────────┬──────────┬──────────┘
                                  │ Yes      │ No
                                  ▼          │
                       ┌────────────────────┐│
                       │  Open Dustbin Lid   ││
                       └────────────┬────────┘│
                                    │         │
                                    ▼         │
                        ┌─────────────────────┐
                        │ Read Moisture Value │
                        └────────────┬────────┘
                                     │
                   ┌─────────────────▼────────────────┐
                   │   Wet Waste ? (Moisture > 500)    │
                   └────────────┬──────────┬───────────┘
                                │ Yes      │ No
                                ▼          ▼
                   ┌──────────────────┐   ┌──────────────────┐
                   │ Servo to WET Bin │   │ Servo to DRY Bin │
                   └────────────┬─────┘   └────────────┬─────┘
                                │                      │
                                └────────────┬─────────┘
                                             ▼
                                ┌──────────────────────────┐
                                │ Close Lid & Reset System │
                                └──────────────────────────┘



