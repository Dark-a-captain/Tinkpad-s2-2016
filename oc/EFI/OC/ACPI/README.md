#如果使用本dsdt.aml卡代码

#则需要自己提取dsdt打补丁

#只需要打快捷键补丁、原生电源补丁、电池补丁

#快捷键将（_Q16  _Q64  _Q65）更换为一下代码：


into method label _Q16 replace_content begin

                {
                    Notify (PS2K, 0x0167)
                    Notify (PS2K, 0x01E7)
                }
end;

into method label _Q64 replace_content begin

                {
                    Notify (PS2K, 0x0165)
                    Notify (PS2K, 0x01E5)
                }
end;

into method label _Q65 replace_content begin

                {
                    Notify (PS2K, 0x0166)
                    Notify (PS2K, 0x01E6)
                }

end

#原生电源补丁代码：

# inject "compatible" with recognized series-100 LPC device-id
into method label _DSM parent_adr 0x001F0000 remove_entry;
into device name_adr 0x001F0000 insert
begin
Method (_DSM, 4, NotSerialized)\n
{\n
    If (LEqual (Arg2, Zero)) { Return (Buffer() { 0x03 } ) }\n
    Return (Package()\n
    {\n
        "compatible", "pci8086,9cc1",\n
    })\n
}\n
end;


#电池补丁：

# Change EC register from 16-bit to 8-bit


into device label EC code_regex ECRC,\s+16 replace_matched begin CRC0,8,CRC1,8 end;
into device label EC code_regex ECAC,\s+16 replace_matched begin CAC0,8,CAC1,8 end;
into device label EC code_regex ECVO,\s+16 replace_matched begin CVO0,8,CVO1,8 end;
into device label EC code_regex SBFC,\s+16 replace_matched begin BFC0,8,BFC1,8 end;
into device label EC code_regex SBDC,\s+16 replace_matched begin BDC0,8,BDC1,8 end;
into device label EC code_regex SBDV,\s+16 replace_matched begin BDV0,8,BDV1,8 end;
into device label EC code_regex SBSN,\s+16 replace_matched begin BSN0,8,BSN1,8 end;


# Change access to EC register (16-bit to 8-bit)

into method label B1B2 remove_entry;
into definitionblock code_regex . insert
begin
Method (B1B2, 2, NotSerialized) { Return(Or(Arg0, ShiftLeft(Arg1, 8))) }\n
end;

into_all method label GBST code_regex \(\^PCI0\.LPCB\.EC\.ECRC, replaceall_matched begin (B1B2 (^PCI0.LPCB.EC.CRC0, ^PCI0.LPCB.EC.CRC1), end;
into_all method label GBST code_regex \(\^PCI0\.LPCB\.EC\.ECAC, replaceall_matched begin (B1B2 (^PCI0.LPCB.EC.CAC0, ^PCI0.LPCB.EC.CAC1), end;
into_all method label GBST code_regex \(\^PCI0\.LPCB\.EC\.ECVO, replaceall_matched begin (B1B2 (^PCI0.LPCB.EC.CVO0, ^PCI0.LPCB.EC.CVO1), end;

into_all method label GBIF code_regex \(\^PCI0\.LPCB\.EC\.SBFC, replaceall_matched begin (B1B2 (^PCI0.LPCB.EC.BFC0, ^PCI0.LPCB.EC.BFC1), end;
into_all method label GBIF code_regex \(\^PCI0\.LPCB\.EC\.SBDC, replaceall_matched begin (B1B2 (^PCI0.LPCB.EC.BDC0, ^PCI0.LPCB.EC.BDC1), end;
into_all method label GBIF code_regex \(\^PCI0\.LPCB\.EC\.SBDV, replaceall_matched begin (B1B2 (^PCI0.LPCB.EC.BDV0, ^PCI0.LPCB.EC.BDV1), end;
into_all method label GBIF code_regex \(\^PCI0\.LPCB\.EC\.SBSN, replaceall_matched begin (B1B2 (^PCI0.LPCB.EC.BSN0, ^PCI0.LPCB.EC.BSN1), end;

# sleep related
into method label _L1D parent_label _GPE code_regex \(\\_SB\.PCI0\.LPC\.EC\.HWAK, replaceall_matched begin (B1B2(\\_SB.PCI0.LPC.EC.WAK0,\\_SB.PCI0.LPC.EC.WAK1), end;
into method label _L1D parent_label \_GPE code_regex \(\\_SB\.PCI0\.LPC\.EC\.HWAK, replaceall_matched begin (B1B2(\\_SB.PCI0.LPC.EC.WAK0,\\_SB.PCI0.LPC.EC.WAK1), end;

# Change EC register from 32-bit to 8-bit
into device label EC code_regex SBCH,\s+32 replace_matched begin BCH0,8,BCH1,8,BCH2,8,BCH3,8 end;

# Change access to EC register (32-bit to 8-bit)
into method label B1B4 remove_entry;
into definitionblock code_regex . insert
begin
Method (B1B4, 4, NotSerialized)\n
{\n
    Store(Arg3, Local0)\n
    Or(Arg2, ShiftLeft(Local0, 8), Local0)\n
    Or(Arg1, ShiftLeft(Local0, 8), Local0)\n
    Or(Arg0, ShiftLeft(Local0, 8), Local0)\n
    Return(Local0)\n
}\n
end;

into_all method label GBIF code_regex \(\^PCI0\.LPCB\.EC\.SBCH, replaceall_matched begin (B1B4 (^PCI0.LPCB.EC.BCH0, ^PCI0.LPCB.EC.BCH1, ^PCI0.LPCB.EC.BCH2, ^PCI0.LPCB.EC.BCH3), end;

# utility methods to read/write buffers from/to EC
into method label RE1B parent_label EC remove_entry;
into method label RECB parent_label EC remove_entry;
into device label EC insert
begin
Method (RE1B, 1, NotSerialized)\n
{\n
    OperationRegion(ERAM, EmbeddedControl, Arg0, 1)\n
    Field(ERAM, ByteAcc, NoLock, Preserve) { BYTE, 8 }\n
    Return(BYTE)\n
}\n
Method (RECB, 2, Serialized)\n
{\n
    ShiftRight(Arg1, 3, Arg1)\n
    Name(TEMP, Buffer(Arg1) { })\n
    Add(Arg0, Arg1, Arg1)\n
    Store(0, Local0)\n
    While (LLess(Arg0, Arg1))\n
    {\n
        Store(RE1B(Arg0), Index(TEMP, Local0))\n
        Increment(Arg0)\n
        Increment(Local0)\n
    }\n
    Return(TEMP)\n
}\n
end;

# deal with 128-bit SBMN
into device label EC code_regex (SBMN,)\s+(128) replace_matched begin BMNX,%2,//%1%2 end;
into method label GBIF code_regex \(\^PCI0\.LPCB\.EC\.SBMN, replaceall_matched begin (^PCI0.LPCB.EC.RECB(0xA0,128), end;

# deal with 128-bit SBDN
into device label EC code_regex (SBDN,)\s+(128) replace_matched begin BDNX,%2,//%1%2 end;
into method label GBIF code_regex \(\^PCI0\.LPCB\.EC\.SBDN, replaceall_matched begin (^PCI0.LPCB.EC.RECB(0xA0,128), end;


#以上代码来源于大神RehabMan