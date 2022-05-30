# import { useFocusEffect } from "@react-navigation/native"
import _ from "lodash"
import moment from "moment"
import React, { useCallback, useEffect, useMemo, useRef, useState } from "react"
import {
  FlatList,
  GestureResponderEvent,
  Image,
  Keyboard,
  LayoutAnimation,
  ListRenderItemInfo,
  Platform,
  RefreshControl,
  Text,
  TouchableOpacity,
  View,
} from "react-native"
import { DateObject } from "react-native-calendars"
import Modal from "react-native-modalbox"
import Icon from "react-native-vector-icons/FontAwesome"
import FontAwesome5 from "react-native-vector-icons/FontAwesome5"

import { getOrderDetail, getOrderHistList } from "/apis/bond"
import {
  GetOrderDetailRes,
  GetOrderHisDetailParams,
  GetOrderHisListParams,
  GetOrderHisListRes,
  OrderHisListRes,
} from "/apis/bond/types"
import { getContractDetail, getContractFilter, getContractList } from "/apis/contract"
import { CANLENDAR2, IC_DROP_DOWN } from "/assets/images"
import { AppIndicator, CalendarPicker, DetailInfo, Indicator, Select } from "/components"
import { AlertDialog } from "/components/AlertDialog"
import { SelectOption } from "/components/Select/types"
import { Colors } from "/configs"
import { numberFormat } from "/configs/numberFormat"
import { FORMAT_DD_MM_YYYY } from "/constant/TimeFormat"
import { CONTRACT, CONTRACT_DETAIL } from "/models/Contract"
import { fontSizer, heightWindow, isIPhoneX, log, responsiveH, responsiveW, width } from "/utils"
import useAccountActive from "/utils/hook/useAccountActive"
import useLanguage from "/utils/hook/useLanguage"
import MultiLanguage from "/utils/MultiLanguage"

import { PickDateProps } from "../Contract/types"
import styles from "./styles"

export const Field = { FromDate: 0, ToDate: 1, BondCode: 2, ExpiredDate: 3 }

const TransactionHistory = () => {
  const [language] = useLanguage()

  const after7days = new Date().setDate(-7)
  const currentDay = moment().format("DD/MM/yyyy")

  // const dayTest = new Date().setDate(-500) // sau test nhớ bỏ

  const [fromTime, setFromTime] = useState<string>(moment(after7days).format("DD/MM/yyyy"))
  const [toTime, setToTime] = useState<string>(moment().format("DD/MM/yyyy"))
  const [showIndex, setShowIndex] = useState<number | null>(null)
  const [detailItem, setDetailItem] = useState<GetOrderDetailRes | null>()
  const [loading, setLoading] = useState<boolean>(true)
  const [alert, setAlert] = useState<string>("")
  const [showResult, setShowResult] = useState<boolean>(false)
  const [refreshing, setRefreshing] = useState<boolean>(false)
  const [isOpenError, setIsOpenError] = useState<boolean>(false)
  const [tableData, setTableData] = useState<Array<GetOrderHisListRes>>()
  const flatListRef = useRef<FlatList>(null)
  const [pageY, setPageY] = useState<number>()
  const [accountActive, setAccountActive] = useAccountActive()

  const [dataSearch, setDataSearch] = useState<any>([])
  const [showModal, setShowModal] = useState(false)
  const DataFilter = ["Tất cả", "Kỳ hạn cố định", "Kỳ hạn linh hoạt", "Outright", "Outright kỳ hạn", "TP TVSI"]
  const [selectedIndex, setSelectedIndex] = useState(-1)
  const [listMaturityDate, setListMaturityDate] = useState<SelectOption[]>([])
  const [contractFilter, setContractFilter] = useState<Array<{ id: number; name: string; issuer: string }>>([])
  const [listFilterProduct, setListFilterProduct] = useState([0])

  const [selectedContract, setSelectedContract] = useState<CONTRACT_DETAIL>()

  const [resContractList, setResContractList] = useState<Array<CONTRACT>>([])
  const [contractList, setContractList] = useState<Array<any>>([])

  const [expiredTime, setExpiredTime] = useState<string>(currentDay)
  const [textInputFilter, setTextInputFilter] = useState("")
  const [editedField, setEditedField] = useState<Array<number>>([])
  const [maturityDateSelected, setMaturityDateSelected] = useState<SelectOption>()
  const [bondSelected, setBondSelected] = useState<SelectOption>()

  useFocusEffect(
    useCallback(() => {
      initOrderHistList()
      const interval = setInterval(
        () => {
          initOrderHistList()
        },
        10000,
        // remoteConfig().getValue("refresh_data_interval").asNumber() * 1000
      )
      return () => {
        setShowIndex(null)
        clearInterval(interval)
      }
    }, [fromTime, toTime]),
  )

  useEffect(() => {
    // LayoutAnimation.easeInEaseOut()
    // setSelectedFilter(4)
    setListFilterProduct([0])
    init()
    return () => {
      setShowModal(false)
      setSelectedContract({})
      setSelectedIndex(-1)
    }
  }, [])
  // useEffect(() => {
  //   const interval = setInterval(() => {
  //     initOrderHistList()
  //   }, remoteConfig().getValue("refresh_data_interval").asNumber() * 1000)
  //   return () => clearInterval(interval)
  // }, [fromTime, toTime])

  const onRefresh = useCallback(() => {
    setRefreshing(true)
    initOrderHistList()
    return () => {
      setShowIndex(null)
    }
  }, [fromTime, toTime])

  const initOrderHistList = async () => {
    const miliStart = Date.parse(moment(`${fromTime} 23:59`, "DD/MM/YYYY HH:mm").format("YYYY-MM-DD"))
    const miliEnd = Date.parse(moment(`${toTime} 23:59`, "DD/MM/YYYY HH:mm").format("YYYY-MM-DD"))
    const miliNow = Date.parse(moment().format("YYYY-MM-DD"))

    if (miliStart > miliEnd) {
      setShowResult(false)
      setAlert(MultiLanguage("Thời gian bắt đầu không được lớn hơn thời gian kết thúc", language.textStatic))
      setIsOpenError(true)
      return
    } else if (miliNow < miliEnd) {
      setShowResult(false)
      setAlert(MultiLanguage("Thời gian kết thúc không được lớn hơn thời gian hiện tại", language.textStatic))
      setIsOpenError(true)
      return
    }
    const params: GetOrderHisListParams = {
      UserId: accountActive.userId,
      AccountNo: accountActive.accountNo,
      FromDate: fromTime,
      ToDate: toTime,
    }
    try {
      setLoading(true)
      const init = await getOrderHistList(params)
      if (!init[0].tradeDate) {
        setShowResult(false)
        setAlert(`Chưa có giao dịch phát sinh\nTừ ngày ${fromTime} đến ngày ${toTime}`)
        return
      }
      setTableData(init)
      setShowResult(true)
      setRefreshing(false)
    } catch (error) {
      log(error)
      setLoading(false)
      setAlert(MultiLanguage("Lỗi! Thử lại sau", language.textStatic))
    } finally {
      setLoading(false)
    }
  }

  const initOrderHistDetail = async (item: OrderHisListRes, index: string) => {
    const params: GetOrderHisDetailParams = {
      UserId: accountActive.userId,
      AccountNo: accountActive.accountNo,
      ContractNo: item.contractNo,
    }
    try {
      setLoading(true)
      const init = await getOrderDetail(params)
      setDetailItem(init)
    } catch (error) {
      log(error)
      setLoading(false)
    } finally {
      setLoading(false)
    }
  }

  const handleShowInfo = (item: OrderHisListRes, index: string) => {
    if (index === showIndex) {
      setShowIndex(null)
      return
    }
    setDetailItem(null)
    initOrderHistDetail(item, index)
    setShowIndex(index)
  }
  //start
  const filterDataSearch = async (value: any) => {
    let arr = [...listFilterProduct]

    if (!listFilterProduct.includes(value)) {
      value !== 0 && _.remove(arr, (e) => e === 0)
      if (value === 0) arr = []
      setListFilterProduct([...arr, value])
    } else if (value !== 0) {
      _.remove(arr, (e) => e === value)
      setListFilterProduct(arr)
    }
  }
  function getDataTable(data: Array<CONTRACT>) {
    // const renderList = data?.map((e) => {
    //   const type =
    //     e?.isRoll && e?.isSell
    //       ? CONTRACT_STATUS.ALL
    //       : e?.isRoll && !e?.isSell
    //       ? CONTRACT_STATUS.EXTEND
    //       : !e?.isRoll && e?.isSell
    //       ? CONTRACT_STATUS.SELL
    //       : CONTRACT_STATUS.NONE
    //   return [e?.contractNo, e?.productTypeName, e?.maturityDate, e?.volume, type]
    // })
    // setContractList(renderList)
    setContractList(data)
  }
  const clearModal = () => {
    handleModal(false)
    setEditedField([])
    setTextInputFilter("")
    setFromTime(moment(after7days).format("DD/MM/yyyy"))
    setToTime(currentDay)
    setExpiredTime(currentDay)
    getDataTable(resContractList)
    setListFilterProduct([0])
    setBondSelected(null!)
    setMaturityDateSelected(null!)
  }

  const onSubmitFilter = (isRefreshData: boolean = false) => {
    {
      console.log("tra cuu", listFilterProduct, fromTime, toTime, bondSelected?.name, maturityDateSelected?.name)
    }
    showModal && !isRefreshData && handleModal(false)
    const resourceList = [...resContractList]
    if (!listFilterProduct.includes(0)) {
      _.remove(resourceList, (e, i) => {
        if (!listFilterProduct.includes(e?.productType)) {
          return e
        }
      })
    }
    _.remove(resourceList, (e, i) => {
      if (!e.bondCode.toUpperCase().includes(textInputFilter.toUpperCase())) {
        return e
      }
    })
    if (editedField.includes(Field.FromDate)) {
      _.remove(resourceList, (e, i) => {
        const isAfter = checkDateIsAfter(fromTime, e?.tradeDate)
        if (isAfter) {
          return e
        }
      })
    }
    if (editedField.includes(Field.ToDate)) {
      _.remove(resourceList, (e, i) => {
        const isAfter = checkDateIsAfter(e?.tradeDate, toTime)
        if (isAfter) {
          return e
        }
      })
    }
    if (editedField.includes(Field.ExpiredDate)) {
      _.remove(resourceList, (e, i) => {
        if (e?.maturityDate !== maturityDateSelected?.name) {
          return e
        }
      })
    }
    getDataTable(resourceList as any)
  }

  const _onSelectExpiredDate = (value: SelectOption) => {
    setMaturityDateSelected(value)
    !editedField.includes(Field.ExpiredDate) && setEditedField([...editedField, ...[Field.ExpiredDate]])
  }

  const _onSelectContract = (value: SelectOption) => {
    setBondSelected(value)
    setTextInputFilter(value.name!)
    !editedField.includes(Field.ExpiredDate) && setEditedField([...editedField, ...[Field.BondCode]])
  }

  function filterOnPress(value: number) {
    if (selectedContract?.contractNo) {
      // LayoutAnimation.easeInEaseOut()
      setSelectedContract({})
      setSelectedIndex(-1)
    }
    value < 6 ? filterDataSearch(value) : handleModal(true)
  }

  const modalDismissKeyboard = () => {
    Keyboard.dismiss()
    setDataSearch([])
  }
  function handleModal(value: boolean) {
    setShowModal(value)
  }

  const handleFromTime = (date: DateObject) => {
    Keyboard.dismiss()
    const currentDate = moment(date.dateString).format(FORMAT_DD_MM_YYYY)
    setFromTime(currentDate)
    const isAfter = checkDateIsAfter(currentDate, toTime)
    isAfter && setToTime(currentDate)
    !editedField.includes(Field.FromDate) && setEditedField([...editedField, ...[Field.FromDate]])
  }

  const handleToTime = (date: DateObject) => {
    setToTime(moment(date.dateString).format("DD/MM/yyyy"))
    !editedField.includes(Field.ToDate) && setEditedField([...editedField, ...[Field.ToDate]])
  }

  async function init() {
    const res = await callApiGetListContract()
    getDataTable(res)

    const dataFilter: any = await getContractFilter({
      UserId: accountActive.userId,
      AccountNo: accountActive.accountNo,
    })
    const list = dataFilter.map((item: any, i: number) => {
      return { id: i, name: item?.bondCode, sub: item?.issuer }
    })
    setContractFilter(list)
    setRefreshing(false)
  }

  async function callApiGetListContract() {
    const dataContractList = await getContractList({
      UserId: accountActive.userId,
      AccountNo: accountActive.accountNo,
    })
    setResContractList(dataContractList)
    getListEndDate(dataContractList)
    return dataContractList
  }

  function getListEndDate(data: Array<CONTRACT>) {
    const listEndDate = data.map((e) => e.maturityDate)
    let date = [...new Set(listEndDate)]

    const sortData = date.sort((i, j) => {
      return moment(i, FORMAT_DD_MM_YYYY).toDate().getTime() - moment(j, FORMAT_DD_MM_YYYY).toDate().getTime()
    })
    const value = sortData.map((e, i) => {
      return { id: i, name: e }
    })
    setListMaturityDate(value)
  }

  const dateTimeElement = (dateTime: string, type: number) => {
    return (
      <View style={styles.triggerElement}>
        <Text
          style={{ color: editedField.includes(type) ? Colors.textColor : Colors.textGray }}
          allowFontScaling={false}
        >
          {dateTime}
        </Text>
        {/* <Ic name={"calendar-alt"} color={"#808080"} size={16} /> */}
        <Image style={{ width: 16, height: 16 }} resizeMode="contain" source={CANLENDAR2} />
      </View>
    )
  }

  const checkDateIsAfter = (date1: string, date2: string) => {
    return moment(date1, FORMAT_DD_MM_YYYY).isAfter(moment(date2, FORMAT_DD_MM_YYYY))
  }

  const sumVolume = (data: any) => {
    let sumall = data.map((item) => item.volume).reduce((prev, curr) => prev + curr, 0)
    return sumall
  }

  useEffect(() => {
    pageY! > heightWindow - responsiveH(248) &&
      flatListRef.current?.scrollToOffset({ offset: pageY! - responsiveH(25), animated: true })
  }, [detailItem, pageY])

  const renderOrderHis = (item: OrderHisListRes, index: string, indexNumber: number) => {
    const showInfo = showIndex === index

    return (
      <View style={{ borderBottomWidth: 0.5, borderBottomColor: "#D6D6D6" }}>
        <TouchableOpacity
          onPress={(event) => handleShowInfo(item, index)}
          style={[styles.rowStyle, { backgroundColor: showInfo ? "#f5f5f5" : "#fff" }]}
          key={index}
        >
          <View style={{ flex: 0.25, paddingVertical: 10 }}>
            <Image
              style={{
                width: responsiveW(10),
                height: responsiveW(15),
                alignSelf: "center",
                // marginRight: responsiveW(14),
                transform: [{ rotate: showInfo ? "180deg" : "0deg" }],
              }}
              resizeMode="contain"
              source={IC_DROP_DOWN}
            />
          </View>
          <View
            style={{
              flexDirection: "column",
              flex: 1.5,
            }}
          >
            <Text
              allowFontScaling={false}
              style={[
                styles.headerText,
                { color: showInfo ? "#ed1c24" : "#333333", fontSize: 15, fontWeight: "500", paddingVertical: 10 },
              ]}
            >
              {item.contractNo || ""}
            </Text>
            <Text allowFontScaling={false} style={[styles.headerText, { paddingVertical: 5 }]}>
              {MultiLanguage("Mã trái phiếu", language.textStatic)}
            </Text>
            <Text allowFontScaling={false} style={[styles.headerText, { paddingVertical: 5 }]}>
              {MultiLanguage("Sản phẩm", language.textStatic)}
            </Text>
            <Text allowFontScaling={false} style={[styles.headerText, { paddingVertical: 5 }]}>
              {MultiLanguage("Số lượng", language.textStatic)}
            </Text>
            <Text
              allowFontScaling={false}
              style={[
                styles.headerText,
                {
                  paddingVertical: 5,
                  color:
                    item?.sideName === MultiLanguage("Bán", language.textStatic)
                      ? "#ED1C24"
                      : item?.sideName === MultiLanguage("Mua", language.textStatic)
                      ? "#238500"
                      : item?.sideName === MultiLanguage("Gia hạn", language.textStatic)
                      ? "#FF6600"
                      : "#333333",
                },
              ]}
            >
              {MultiLanguage(item.sideName, language.textStatic) || ""}
            </Text>
          </View>

          <View
            style={{
              flexDirection: "column",
              flex: 1,
              paddingRight: 10,
            }}
          >
            <Text allowFontScaling={false} style={{ paddingVertical: 10 }}>
              {MultiLanguage(" ", language.textStatic)}
            </Text>
            <Text
              allowFontScaling={false}
              style={[styles.bodyText, { color: "#333333", paddingVertical: 5, textAlign: "right" }]}
            >
              {item.bondCode || ""}
            </Text>
            <Text
              allowFontScaling={false}
              style={[styles.bodyText, { paddingVertical: 5, color: "#333333", textAlign: "right" }]}
            >
              {item.productTypeName || ""}
            </Text>
            <Text
              allowFontScaling={false}
              style={[styles.bodyText, { color: "#333333", paddingVertical: 5, textAlign: "right" }]}
            >
              {numberFormat(item.volume) || ""}
            </Text>
            <Text
              allowFontScaling={false}
              style={[
                styles.bodyText,
                {
                  paddingVertical: 5,
                  marginBottom: 10,
                  textAlign: "right",
                  color:
                    item?.statusName === MultiLanguage("Hoàn thành", language.textStatic)
                      ? "#ed1c24"
                      : item?.statusName === MultiLanguage("Đã xác nhận", language.textStatic)
                      ? "#238500"
                      : item?.statusName === MultiLanguage("Hủy lệnh", language.textStatic)
                      ? "#FF6600"
                      : "#333333",
                },
              ]}
            >
              {MultiLanguage(item.statusName, language.textStatic) || ""}
            </Text>
          </View>
        </TouchableOpacity>
        {showInfo && <DetailInfo detailItem={detailItem!} />}
      </View>
    )
  }

  const renderItem = ({ item, index }: ListRenderItemInfo<GetOrderHisListRes>) => {
    return (
      <>
        <View
          key={index}
          style={{
            backgroundColor: "#F5F5F5",
            paddingVertical: responsiveH(7),
            paddingHorizontal: responsiveW(43),
            flexDirection: "row",
            alignItems: "center",
          }}
        >
          <Text allowFontScaling={false} style={styles.bodyText}>
            {item.tradeDate === moment(new Date()).format("DD/MM/yyyy")
              ? MultiLanguage("Hôm nay", language.textStatic)
              : item?.tradeDate || ""}
          </Text>
        </View>
        {item.orderHistList.map((item, i) => renderOrderHis(item, `${index}-${i}`, index))}
      </>
    )
  }
  const renderFilterModal = () => {
    const PickDate = (props: PickDateProps) => (
      <View
        style={{
          marginBottom: responsiveH(17),
        }}
      >
        <Text style={styles.bodyText} allowFontScaling={false}>
          {props.label}
        </Text>
        <CalendarPicker
          minDate={props.minDate}
          maxDate={props.maxDate}
          containerStyle={styles.calendarWrap}
          triggerCalendar={props.triggerCalendar}
          onDayPress={props.onDayPress}
          _onPressIn={modalDismissKeyboard}
        />
      </View>
    )
    return (
      <Modal
        position="bottom"
        style={[styles.modal]}
        onClosed={() => handleModal(false)}
        isOpen={showModal}
        keyboardTopOffset={0}
        swipeToClose={false}
      >
        <View style={{ flexDirection: "row", justifyContent: "center" }}>
          <View>
            <Text style={styles.filterText} allowFontScaling={false}>
              {MultiLanguage("Tra cứu theo ", language.textStatic)}
            </Text>
          </View>
          <View>
            <Text style={[styles.filterText, { color: "#ED1C24" }]} allowFontScaling={false}>
              {MultiLanguage("bộ lọc", language.textStatic)}
            </Text>
          </View>
        </View>
        <View
          style={{
            flexWrap: "wrap",
            paddingBottom: 10,
            paddingHorizontal: isIPhoneX ? responsiveW(20) : Platform.OS == "ios" ? responsiveW(6) : responsiveW(20),
            flexDirection: "row",
            width: "100%",
            backgroundColor: "white",
            marginHorizontal: responsiveW(8),
          }}
        >
          {DataFilter.map((e, i) => {
            return (
              <TouchableOpacity
                onPress={() => filterOnPress(i)}
                key={i}
                style={[styles.filterItem, i === 6 && { borderRightWidth: 0, marginRight: 0 }]}
              >
                <Text
                  style={[
                    styles.textFilter,
                    listFilterProduct.includes(i)
                      ? { color: Colors.red, borderColor: "red", borderWidth: 1 }
                      : { color: "#A3A3A3", borderColor: "#A3A3A3", borderWidth: 1 },
                    { fontSize: fontSizer(14) },
                  ]}
                  allowFontScaling={false}
                >
                  {MultiLanguage(e, language.textStatic)}
                </Text>
              </TouchableOpacity>
            )
          })}
        </View>
        <TouchableOpacity
          onPress={modalDismissKeyboard}
          activeOpacity={1}
          style={{ paddingHorizontal: responsiveW(19) }}
        >
          <View style={{ flexDirection: "row", justifyContent: "space-between" }}>
            <PickDate
              label={MultiLanguage("Từ ngày", language.textStatic)}
              onDayPress={handleFromTime}
              maxDate={new Date()}
              triggerCalendar={dateTimeElement(fromTime, Field.FromDate)}
            />
            <PickDate
              label={MultiLanguage("Đến ngày", language.textStatic)}
              minDate={editedField.includes(Field.FromDate) ? moment(`${fromTime}`, "DD/MM/YYYY").format() : undefined}
              maxDate={new Date()}
              onDayPress={handleToTime}
              triggerCalendar={dateTimeElement(toTime, Field.ToDate)}
            />
          </View>

          <View
            style={{
              marginBottom: responsiveH(17),
            }}
          >
            <Text style={styles.bodyText} allowFontScaling={false}>
              {MultiLanguage("Mã trái phiếu", language.textStatic)}
            </Text>
            <Select
              options={contractFilter}
              label={bondSelected?.name}
              onSelect={_onSelectContract}
              containerStyle={styles.containerStyle}
              triggerStyle={styles.triggerStyle}
              menuStyle={{ ...styles.menuStyle, height: responsiveH(250) }}
              labelStyle={[styles.labelStyle, editedField.includes(Field.BondCode) && { color: Colors.black }] as any}
              textMenuStyle={
                {
                  // fontWeight: "bold",
                  // paddingBottom: 2,
                }
              }
              hideLeft={true}
              hideImageLeft={true}
              colorRight={Colors.black}
              showSub={true}
            />
          </View>
          <View>
            <Text style={styles.bodyText} allowFontScaling={false}>
              {MultiLanguage("Ngày đáo hạn", language.textStatic)}
            </Text>
            <Select
              options={listMaturityDate}
              label={maturityDateSelected?.name}
              onSelect={_onSelectExpiredDate}
              containerStyle={styles.containerStyle}
              triggerStyle={styles.triggerStyle}
              menuStyle={styles.menuStyle}
              labelStyle={
                [styles.labelStyle, editedField.includes(Field.ExpiredDate) && { color: Colors.black }] as any
              }
              hideLeft={true}
              hideImageLeft={true}
              colorRight={Colors.textColor}
            />
          </View>

          <View
            style={{
              flexDirection: "row",
              alignSelf: "center",
              marginTop: responsiveH(20),
            }}
          >
            <TouchableOpacity
              onPress={clearModal}
              style={{
                borderRadius: 50,
                paddingVertical: responsiveH(12),
                paddingHorizontal: responsiveW(30),
                backgroundColor: "#333333",
                marginRight: responsiveW(15),
              }}
              children={
                <Text
                  style={styles.buttonText}
                  children={MultiLanguage("Hủy bỏ", language.textStatic)}
                  allowFontScaling={false}
                />
              }
            />
            <TouchableOpacity
              onPress={() => onSubmitFilter(false)}
              style={{
                borderRadius: 50,
                backgroundColor: "#ED1C24",
              }}
              children={
                <Text
                  style={[
                    styles.buttonText,
                    { paddingVertical: responsiveH(12), paddingHorizontal: responsiveW(30), color: "#ffffff" },
                  ]}
                  allowFontScaling={false}
                  children={MultiLanguage("Tra cứu", language.textStatic)}
                />
              }
            />
          </View>
        </TouchableOpacity>
      </Modal>
    )
  }

  const renderEmpty = useMemo(() => {
    return (
      <View
        style={{
          justifyContent: "flex-start",
          marginTop: responsiveH(20),
          alignItems: "center",
          flex: 1,
        }}
      >
        <Text
          allowFontScaling={false}
          style={{
            fontSize: fontSizer(14),
            color: "#333",
            textAlign: "center",
          }}
        >
          {alert}
        </Text>
        <Image source={CANLENDAR2} style={{ width: 25, height: 25, marginTop: 5 }} />
      </View>
    )
  }, [alert])

  const timeElement = useMemo(
    () => (dateTime: string) => {
      return (
        <View style={styles.triggerElement}>
          <Text allowFontScaling={false}>{dateTime}</Text>
          {/* <FontAwesome5 name={"calendar-alt"} color={"#808080"} size={16} /> */}
          <Image style={{ width: 16, height: 16 }} resizeMode="contain" source={CANLENDAR2} />
        </View>
      )
    },
    [fromTime, toTime],
  )

  // const handleFromTime = (date: DateObject) => {
  //   setFromTime(moment(date.dateString).format("DD/MM/yyyy"))
  // }

  // const handleToTime = (date: DateObject) => {
  //   setToTime(moment(date.dateString).format("DD/MM/yyyy"))
  // }

  return (
    <View
      style={{
        flex: 1,
        backgroundColor: "#f5f5f5",
      }}
    >
      <View
        style={{
          backgroundColor: "#fff",
          paddingVertical: responsiveW(10),
          paddingHorizontal: responsiveW(15),
          borderBottomWidth: 0.5,
          borderBottomColor: "#D6D6D6",
        }}
      >
        <View
          style={{
            flexDirection: "row",
            backgroundColor: "white",
            justifyContent: "flex-end",
          }}
        >
          <TouchableOpacity onPress={() => handleModal(true)} activeOpacity={1} style={styles.filterItem}>
            <Text style={[styles.bodyText, { color: "#333", marginTop: 5, marginRight: 3 }]} allowFontScaling={false}>
              {MultiLanguage("Tra cứu theo bộ lọc", language.textStatic)}
            </Text>
            <Icon size={20} color={Colors.black} name="filter" />
          </TouchableOpacity>
        </View>
      </View>

      {/* <View
        style={{
          backgroundColor: "#fff",
          flexDirection: "row",
          justifyContent: "space-between",
          paddingVertical: responsiveH(10),
          paddingHorizontal: responsiveW(30),
        }}
        children={
          <>
            <View
              style={{
                flexDirection: "column",
                width: "45%",
              }}
            >
              <Text allowFontScaling={false} style={[styles.bodyText, { marginBottom: responsiveH(10) }]}>
                {MultiLanguage("Từ ngày", language.textStatic)}
              </Text>
              <CalendarPicker
                maxDate={new Date()}
                triggerCalendar={timeElement(fromTime)}
                onDayPress={handleFromTime}
              />
            </View>
            <View
              style={{
                flexDirection: "column",
                width: "45%",
              }}
            >
              <Text allowFontScaling={false} style={[styles.bodyText, { marginBottom: responsiveH(10) }]}>
                {MultiLanguage("Đến ngày", language.textStatic)}
              </Text>
              <CalendarPicker
                minDate={moment(`${fromTime} 23:59`, "DD/MM/YYYY HH:mm").format()}
                maxDate={new Date()}
                triggerCalendar={timeElement(toTime)}
                onDayPress={handleToTime}
              />
            </View>
          </>
        }
      /> */}
      <View
        style={{
          height: 2,
          backgroundColor: "#f5f5f5",
        }}
      />

      {/*<View
        style={{
          flexDirection: "row",
          paddingHorizontal: responsiveW(20),
          paddingVertical: responsiveH(10),
          backgroundColor: "#ffffff",
        }}
      >
          <View style={{flex:0.25}}></View>
          <View style={{flex:3,marginLeft:11}}>
               <Text
           allowFontScaling={false} 
                  style={styles.headerText}
                  children={MultiLanguage("Số HĐ", language.textStatic)}
              />
               <Text
           allowFontScaling={false}  style={styles.headerText}>
                  {MultiLanguage("Mã trái phiếu", language.textStatic)}
              </Text>
          </View>
          <View style={{flex:3, textAlign:'left'}}>
               <Text
           allowFontScaling={false} 
                  style={styles.headerText}
                  children={MultiLanguage("Sản phẩm", language.textStatic)}
              />
               <Text
           allowFontScaling={false}  style={styles.headerText}>
                  {MultiLanguage("Số lượng", language.textStatic)}
              </Text>
          </View>
          <View style={{flex:1.75}}>
               <Text
           allowFontScaling={false} 
                  style={styles.headerText}

                  children={MultiLanguage("Loại GD", language.textStatic)}
              />
               <Text
           allowFontScaling={false}  style={styles.headerText}>
                  {MultiLanguage("Trạng thái", language.textStatic)}
              </Text>
          </View>


      </View>*/}
      {/* {console.log("tabledata", tableData)} */}
      {(tableData && tableData.length != 0 && showResult && (
        <FlatList
          showsVerticalScrollIndicator={false}
          bounces={true}
          data={tableData}
          ref={flatListRef}
          renderItem={renderItem}
          keyExtractor={(item, index) => index.toString()}
          scrollEventThrottle={16}
          refreshControl={<RefreshControl refreshing={refreshing} onRefresh={onRefresh} />}
        />
      )) ||
        renderEmpty}
      {renderFilterModal()}

      <AlertDialog isOpen={isOpenError} handleClose={() => setIsOpenError(false)} errorDetail={alert} />
      {loading && <AppIndicator />}
    </View>
  )
}

export default TransactionHistory
