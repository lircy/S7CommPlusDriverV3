# Siemens S7CommPlus Communication Driver (Testing Phase)

**A .NET-based communication driver for Siemens S7-1200/1500 PLCs without TLS dependency**

## Overview
This communication driver enables secure interaction with **Siemens S7-1200 and S7-1500 PLCs** without requiring TLS configuration. It provides symbolic variable access and batch operations using modern .NET Framework infrastructure.

## Technical Specifications
| Component          | Specification                          |
|--------------------|----------------------------------------|
| Target Framework   | .NET Framework 4.7.2                   |
| Core Protocol      | S7CommPlus                             |
| Security Model     | Non-TLS communication support          |

## Key Features
- 🔍 Online browsing of global symbolic variable tree
- 📥 Batch read operations via symbolic variable names
- 📤 Value writing through symbolic variables
- ⚡ Direct communication with S7-1200/1500 PLCs

## Implementation Foundation
This project integrates and extends the following critical open-source components:

1. **Communication Core**  
   [`S7CommPlusDriver`](https://github.com/thomas-v2/S7CommPlusDriver)  
   *Handles all low-level protocol operations*

2. **Authentication System**  
   [`HarpoS7`](https://github.com/bonk-dev/HarpoS7)  
   *Implements PLC security handshake algorithms*

## Getting Started

### Prerequisites
- Visual Studio 2019+
- .NET Framework 4.7.2 Developer Pack
- Siemens PLC with enabled PUT/GET communication

### Basic Usage
```csharp
class Program
    {
        static void Main(string[] args)
        {
            string HostIp = "192.168.0.250";
            string Password = "";
            int res;
            List<ItemAddress> readlist = new List<ItemAddress>();
            Console.WriteLine("Main - START");
            // Als Parameter lässt sich die IP-Adresse übergeben, sonst Default-Wert von oben
            if (args.Length >= 1) {
                HostIp = args[0];
            }
            // Als Parameter lässt sich das Passwort übergeben, sonst Default-Wert von oben (kein Passwort)
            if (args.Length >= 2) {
                Password = args[1];
            }
            Console.WriteLine("Main - Versuche Verbindungsaufbau zu: " + HostIp);

            S7CommPlusConnection conn = new S7CommPlusConnection();
            res = conn.Connect(HostIp, Password);
            if (res == 0)
            {
                Console.WriteLine("Main - Connect fertig");

                #region Variablenhaushalt browsen
                Console.WriteLine("Main - Starte Browse...");
                // Variablenhaushalt auslesen
                List<VarInfo> vars = new List<VarInfo>();
                res = conn.Browse(out vars);
                Console.WriteLine("Main - Browse res=" + res);
                #endregion
                List<VarInfo> vars_ = vars.GetRange(0,1000);
                #region Werte aller Variablen einlesen
                Console.WriteLine("Main - Lese Werte aller Variablen aus");

                List<PlcTag> taglist = new List<PlcTag>();
                PlcTags tags = new PlcTags();

                foreach (var v in vars_)
                {
                    taglist.Add(PlcTags.TagFactory(v.Name, new ItemAddress(v.AccessSequence), v.Softdatatype));
                }
                foreach (var t in taglist)
                {
                    tags.AddTag(t);
                }
                if (res == 0)
                {
                    Console.WriteLine("====================== VARIABLENHAUSHALT ======================");

                    string formatstring = "{0,-80}{1,-30}{2,-20}{3,-20}";
                    Console.WriteLine(String.Format(formatstring, "SYMBOLIC-NAME", "ACCESS-SEQUENCE", "TYP", "QC: VALUE"));
                    for (int i = 0; i < vars_.Count; i++)
                    {
                        string s;
      
                        s = String.Format(formatstring, taglist[i].Name, taglist[i].Address.GetAccessString(), Softdatatype.Types[taglist[i].Datatype], taglist[i].ToString());
                        Console.WriteLine(s);
                    }
                }
                #endregion
                Console.WriteLine("按任意键开始读");
                Console.ReadKey();
                while (res == 0)
                {
                    System.Diagnostics.Stopwatch stopwatch = new System.Diagnostics.Stopwatch();
                    stopwatch.Start();
                    res = tags.ReadTags(conn);
                    stopwatch.Stop();
                    long ms = stopwatch.ElapsedMilliseconds;
                    if (res == 0)
                    {
                        string header = $"读取{vars_.Count}个变量耗时{ms}毫秒";
                        Console.WriteLine(header);
                    }
                }
                
                #region Werte aller Variablen einlesen
                Console.WriteLine("Main - Lese Werte aller Variablen aus");

                foreach (var v in vars)
                {
                    readlist.Add(new ItemAddress(v.AccessSequence));
                }
                List<object> values = new List<object>();
                List<UInt64> errors = new List<UInt64>();

                
                // Fehlerhafte Variable setzen
                //readlist[2].LID[0] = 123;
                res = conn.ReadValues(readlist, out values, out errors);
                #endregion


                #region Variablenhaushalt mit Werten ausgeben

                if (res == 0)
                {
                    Console.WriteLine("====================== VARIABLENHAUSHALT ======================");

                    // Liste ausgeben
                    string formatstring = "{0,-80}{1,-30}{2,-20}{3,-20}";
                    Console.WriteLine(String.Format(formatstring, "SYMBOLIC-NAME", "ACCESS-SEQUENCE", "TYP", "VALUE"));
                    for (int i = 0; i < vars.Count; i++)
                    { 
                        string s = String.Format(formatstring, vars[i].Name, vars[i].AccessSequence, Softdatatype.Types[vars[i].Softdatatype], values[i]);
                        Console.WriteLine(s);
                    }
                    Console.WriteLine("===============================================================");
                }
                #endregion

                /*
                #region Test: Wert schreiben
                List<PValue> writevalues = new List<PValue>();
                PValue writeValue = new ValueInt(8888);
                writevalues.Add(writeValue);
                List<ItemAddress> writelist = new List<ItemAddress>();
                writelist.Add(new ItemAddress("8A0E0001.F"));
                errors.Clear();
                res = conn.WriteValues(writelist, writevalues, out errors);
                Console.WriteLine("res=" + res);
                #endregion
                */

                /*
                #region Test: Absolutadressen lesen
                // Daten aus nicht "optimierten" Datenbausteinen lesen
                readlist.Clear();
                ItemAddress absAdr = new ItemAddress();
                absAdr.SetAccessAreaToDatablock(100); // DB 100
                absAdr.SymbolCrc = 0;

                absAdr.AccessSubArea = Ids.DB_ValueActual;
                absAdr.LID.Add(3);  // LID_OMS_STB_ClassicBlob
                absAdr.LID.Add(0);  // Blob Start Offset, Anfangsadresse
                absAdr.LID.Add(20); // 20 Bytes

                readlist.Add(absAdr);

                values.Clear();
                errors.Clear();

                res = conn.ReadValues(readlist, out values, out errors);
                Console.WriteLine(values.ToString());
                #endregion
                */


                #region Test: SPS in Stopp setzen
                Console.WriteLine("Setze SPS in STOP...");
                conn.SetPlcOperatingState(1);
                Console.WriteLine("Taste drücken um wieder in RUN zu setzen...");
                Console.ReadKey();
                Console.WriteLine("Setze SPS in RUN...");
                conn.SetPlcOperatingState(3);
                #endregion

                conn.Disconnect();
            }
            else
            {
                Console.WriteLine("Main - Connect fehlgeschlagen!");
            }
            Console.WriteLine("Main - ENDE. Bitte Taste drücken.");
            Console.ReadKey();
        }
    }
```

## Current Status
```diff
+ Functional communication established
+ Symbolic variable operations working
- Stability testing ongoing
- Edge case validation pending
- Production hardening required
```

## Contribution & Feedback
We welcome issue reports and testing feedback during this development phase.  
Please create detailed tickets including:
- PLC firmware version
- Reproduction steps
- Error logs

## License
[MIT](LICENSE) (Compatible with upstream dependencies)

---
